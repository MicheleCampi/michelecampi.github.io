---
layout: post
title: "NVIDIA's KV-router isn't faster. Under load it drops requests — and that's the design."
date: 2026-06-27
categories: observability systems-engineering llm-inference
tags: rust llm inference dynamo kv-router vllm routing benchmarking inferscope aiperf
---
NVIDIA's Dynamo ships a KV-aware router. The documentation shows it beating round-robin, and the mechanism is intuitive: route each request to the worker most likely to already hold its prefix in cache, and you save redundant prefill. I wanted to know how much — not at a single operating point, which is what the guide shows, but across a scaling curve. Does the benefit grow with the number of workers? Stay flat? Something else?

The answer turned out to be "something else," and it took me through three wrong explanations before I found the one the data actually supports. The short version: the KV-router is not "faster" in any honest sense. Under load it sheds requests — it returns 503s to about one in seven callers — and that is precisely how it keeps latency low for everyone else. Round-robin, by contrast, accepts everything and lets latency collapse. This is a trade-off, not a win, and it only exists near saturation. Add capacity and it vanishes.

Here is how I got there.

## The setup

I ran an A/B on a single 8×A100-SXM4-40GB box. Both arms are identical Dynamo deployments — same model (Qwen3-8B), same workers, same `--block-size 64 --max-model-len 16384 --enforce-eager`. The only thing that differs between arms is the frontend flag: `--router-mode round-robin` (I'll call it OFF) versus `--router-mode kv` (ON). Everything else is left at defaults. That detail matters later.

The workload is the Mooncake trace — a real production trace with genuine prefix sharing, which is the thing the KV-router is supposed to exploit. I filtered it to requests with input length ≤ 16384 (21,167 of the 23,608 lines survive) and sliced the first 1,000. The driver is AIPerf replaying the trace on a fixed schedule, so every run sees the same arrival pattern; the three runs per cell measure system variance, not different inputs.

The scaling curve is the point of the whole exercise: I ran the same A/B at N=2, N=4, and N=8 workers, using `CUDA_VISIBLE_DEVICES` to carve the 8-GPU box into the configuration I wanted.

Measurement came from two sides, deliberately. On the client side, AIPerf reports what the caller experiences: time-to-first-token, inter-token latency, throughput, and — it turns out crucially — which requests got an error instead of an answer. On the server side I ran [inferscope](https://github.com/MicheleCampi/inferscope), a profiler I built, in `--sample-only` mode: it attaches to the running worker processes and samples per-device GPU utilisation, power, and memory straight from NVML, on a fixed interval, without driving any load itself. AIPerf tells you what the clients felt; inferscope tells you what the GPUs were actually doing at the same wall-clock moment. The reason I wanted both is that a benchmark which only listens to the client will believe the client's story — and as it turned out, the client's story here was misleading in a way only the server-side numbers exposed. The gap between the two views is where this whole result lives.

## The result that looked like a win

At N=2, the KV-router posts a TTFT p50 of 2.6 seconds against round-robin's 12.1 seconds. A 78% reduction. The average tells the same story: 4.5s versus 16.9s. If I had stopped there — and the single-operating-point version of this benchmark would have stopped there — I would have written "the KV-router cuts TTFT by roughly 75%" and moved on.

| N | arm | TTFT p50 | TTFT avg |
|---|-----|----------|----------|
| 2 | round-robin | 12,125 ms | 16,946 ms |
| 2 | kv-router   | 2,634 ms  | 4,541 ms  |

But the per-device GPU numbers from inferscope didn't fit the story, and that mismatch is the only reason the rest of this post exists. At N=2, round-robin ran the two GPUs at 81% mean utilisation; the KV-router ran them at 69%. Sit with that for a second: the KV-router was serving requests almost five times faster on TTFT while using *less* of the hardware. The cache-reuse story can explain part of it — reusing a prefix means less prefill, less work, lower utilisation. But "serves faster while doing less work" is also exactly what you'd see if the faster arm were quietly doing *fewer requests*, and an average can't tell those two apart. If I'd only had the client's TTFT, I'd have taken the win. The server-side number is what made me distrust it and go to the per-request data.

## Three wrong explanations

My first guess was load imbalance: maybe the KV-router, by pinning prefix-sharing requests to the same workers, overloads some GPUs and starves others, and the averages hide it. inferscope has per-device data, so this was directly testable. It was wrong. The utilisation skew between GPUs is marginal in both arms — standard deviation of 2.0 points for round-robin, 2.7 for the KV-router. Real, but nowhere near enough to explain a 78% latency gap. Imbalance is not the mechanism.

My second guess used the per-request worker IDs in AIPerf's output. That was also wrong, in an instructive way: AIPerf's `worker_id` field is always 32 distinct values regardless of N — it identifies AIPerf's own load-generation workers, not the Dynamo backend that served the request. The client simply does not record which GPU answered. A whole afternoon's analysis built on that field would have been meaningless; checking what it actually counted, before trusting it, is the only reason I caught it.

The third guess was closer but still wrong. I'd read in the Dynamo source that the KV-router has an admission-control path keyed on `max_queued_isl_tokens` — a backpressure threshold on queued input tokens. Plausible! Except the 503s in my data carry the message "All workers are busy, please retry later," and the token-backpressure path emits a different message ("router backpressure: queued_isl_tokens=…"). Wrong code path. The lesson that kept repeating: a plausible mechanism that matches the behaviour is not the same as the mechanism that produced it.

## What the data actually shows

The thing I had been stepping over was sitting in the per-request records the whole time. At N=2, the KV-router doesn't complete 1,000 requests per run. It completes 897, then 851, then 823 — and the missing ones aren't slow, they're *gone*. Each carries an explicit error: HTTP 503, `ResourceExhausted: All workers are busy, please retry later`. Aggregated over three runs:

| N | arm | sent | completed | failed | fail % |
|---|-----|------|-----------|--------|--------|
| 2 | round-robin | 3000 | 3000 | 0   | 0.0%  |
| 2 | kv-router   | 3000 | 2571 | 429 | 14.3% |
| 4 | round-robin | 3000 | 3000 | 0   | 0.0%  |
| 4 | kv-router   | 3000 | 3000 | 0   | 0.0%  |
| 8 | round-robin | 3000 | 3000 | 0   | 0.0%  |
| 8 | kv-router   | 3000 | 3000 | 0   | 0.0%  |

The failures exist in exactly one cell: N=2, KV-router. Zero everywhere else — both arms, every other scale, all three runs each. And they grow run over run at N=2: 10.3%, 14.9%, 17.7%.

This reframes the headline number entirely. The KV-router's "78% lower TTFT" is computed over completed requests — and 14% of requests didn't complete. The advantage is partly survivorship: the requests that would have been slowest under pressure don't show up as slow, they show up as 503s, and 503s have no TTFT to average in.

To see what each arm is really doing, look at how TTFT evolves *within* a run. Round-robin at N=2 is a textbook unbounded queue: bucket the requests by arrival time into deciles and the median TTFT climbs monotonically — 746 ms in the first tenth, 9 seconds by the middle, 39 seconds by the end. The system is saturated, every request waits behind a longer line than the last, and nobody is turned away. The KV-router's TTFT also rises but never runs away: it peaks around 13 seconds and dips back when the queue drains, because instead of queueing the overflow it rejects it.

So the two routers implement opposite policies for the same overload. Round-robin: admit everything, queue, degrade — 100% completion, but tail latency collapses to 39+ seconds. KV-router: shed the overflow with 503s, protect the latency of whoever gets in — 2.6s median, but 14% turned away. Neither is "better" in the abstract. They're different points on a latency-versus-completeness curve.

## Whose decision is this, architecturally?

This is the question that decides whether the result means anything. If the 503s came from something I configured — a queue bound I set, a threshold I passed — then I'd have measured my own setup, not Dynamo. So I traced it to the source, at the exact tag the container runs (`v1.2.0`), not a newer checkout where defaults might differ.

The chain is gated entirely on `--router-mode`. When you select `kv`, the frontend creates a `KvWorkerMonitor` (`lib/llm/src/discovery/watcher.rs`) that tracks each worker's busy state and feeds it to the router, which is built with fault detection enabled. Under load, when the monitor reports no free workers, the router rejects with the 503 I saw (`lib/runtime/src/pipeline/network/egress/push_router.rs`). Select `round-robin` and that monitor is never created: the router never sees a worker as "busy," so it never rejects — it just hands the request to the next worker in rotation and lets the queue grow. The source even comments on the wiring: the monitor must share the router's client so busy-state updates are visible.

In other words: load-shedding is not a tuning knob I turned. It is what the KV-router *is*, by construction, with default settings. Anyone can verify this by reading those two files at tag `v1.2.0`. I passed only `--router-mode`; everything else was default.

## The curve is the finding

Here is why the scaling curve mattered and a single operating point would have lied. The entire effect — the 503s, the latency gap, the whole trade-off — exists only at N=2, where two workers cannot keep up with the trace's arrival rate and the system is genuinely saturated. At N=4 and N=8 there are no failures in either arm, and the KV-router gives no latency benefit at all; if anything it's slightly worse (N=8 TTFT p50: 339 ms with KV versus 197 ms round-robin), the small cost of affinity routing when there's nothing to gain from it.

So the KV-router's value is not a property of the router. It's a property of the *operating regime*. Near saturation it buys you latency at the cost of completeness. Once you have enough capacity that you're no longer prefill-bound, it buys you nothing and costs a little. The crossover sits between N=2 and N=4 for this workload.

The practical takeaway is narrower and more useful than "use the KV-router, it's faster." It's: if you're over-provisioned, the KV-router does nothing for you and adds overhead. If you're running near your limit, it will keep your served latency low — by returning 503s to a chunk of your traffic. You need to know that's the behaviour, because "lower TTFT" and "drops 14% of requests" are the same sentence here, and a dashboard that only plots TTFT over completed requests will show you the first half and hide the second.

## What I can't claim

I measured the client's view and traced the code path; I did not capture the workers' internal state at the moment of rejection — the instance was ephemeral and the logs went with it. I'm asserting the client-observed 503 and the mechanism in source, not the per-worker queue depths at failure time. The failure rate climbs across the three ON runs (10% → 18%); `--router-reset-states` was set only on the KV arm, and I haven't isolated whether the reset or accumulated pressure drives the climb. And the source reading is pinned to v1.2.0, the version that produced the data; later versions may differ. The raw results, the analysis scripts, and the full evidence chain are in the [repo](https://github.com/MicheleCampi/dynamo-kv-router-ab) if you want to check my work — or find the fourth wrong explanation I haven't noticed yet.
