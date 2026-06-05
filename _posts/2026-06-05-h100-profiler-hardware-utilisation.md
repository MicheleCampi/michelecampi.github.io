---
layout: post
title: "The profiler had to teach me about the hardware. The hardware taught me about the profiler."
date: 2026-06-05
categories: observability systems-engineering llm-inference
tags: rust llm profiling observability gpu h100 inferscope
---

I was running my own profiler against an inference server on a rented L4. The report told me the server was using 2 MiB of RAM. The model on disk was 4.7 GB.

That was the first sample I looked at, and it was clearly wrong. What I did about it, and what running the same thing on an H100 a week later taught me about hardware I thought I understood, is the story of this post.

## What inferscope is, briefly

[inferscope](https://github.com/MicheleCampi/inferscope) is a Rust profiler I built for LLM inference engines. It sits between two tools that, on their own, leave a gap. `nvidia-smi` shows the hardware but doesn't know your inference is happening — it just reports VRAM and SM utilisation against wall-clock time. The PyTorch profiler knows the inference is happening but is Python-only, has measurable overhead, and stops at the framework boundary. Neither tells you, on the same shared clock, what the model server is doing on CPU and on GPU **per token**.

inferscope drives an OpenAI-compatible engine through its HTTP API, captures per-token timing from the streamed response, and samples `/proc/<pid>` and (since v0.2.0) NVML in parallel. The output is one JSON document — and one plain-text report — that correlates token arrivals with the host process's RSS, CPU, threads, and the GPU's VRAM, SM utilisation, power draw, temperature. The architecture is a five-crate Cargo workspace; everything is Apache-licensed and runs on stable Rust.

At the time of the L4 run I'm about to describe, inferscope was at v0.2.0. NVML sampling had just landed. I'd validated against a CPU-only llama.cpp baseline (Qwen 2.5 0.5B Q4: TTFT 25 ms, 82.7 tokens/s, 588 MiB RSS). The GPU run was supposed to be the second leg of that validation.

## The 2-MiB report

The L4 setup was the kind of thing anyone would do: rent an L4 pod on RunPod (~$0.45/hr), bring up `llama-server` from `llama.cpp` b9165 with the same Qwen 0.5B Q4 model, point inferscope at it. The shell command was the obvious one:

```bash
./llama-server --model qwen2.5-0.5b-instruct-q4_k_m.gguf \
  --n-gpu-layers 999 --port 8080 &
SERVER_PID=$!

inferscope --endpoint http://127.0.0.1:8080 \
  --model qwen2.5-0.5b-instruct \
  --prompt "..." \
  --max-tokens 80 \
  --pid $SERVER_PID --gpu --json > report.json
```

The report came back. The timing section looked great: TTFT 13 ms, 381 tokens/s. GPU section looked great: VRAM 1.34 GB, SM peak 91%, mean 58%, power 37–39 W on a 72 W TDP. The L4 was doing more than half its power envelope of work, which felt right for a small model.

The resource section did not look right.

```
Process resource usage (16 samples)
  RSS                peak 2 MiB   mean 2 MiB
  CPU utilization    mean 0%
  Threads            1 throughout
```

A first instinct said: the model is on the GPU, so host RSS being low is fine. That instinct lasted about ten seconds, because 2 MiB is not "low for a host-resident inference server" — it is 0.4% of what the actual `llama-server` process should be holding, which is hundreds of MiB of mmap'd model file, tokenizer buffers, KV cache fallback, HTTP server state. One thread is also not "low" — it is wrong. `llama-server` spins up dozens.

`nvidia-smi` confirmed the model was loaded and the GPU was working. So the inference was happening. inferscope was sampling — sixteen successful reads of `/proc/$SERVER_PID/{status,stat}`. It just wasn't sampling the right process.

A `pgrep` on the parent told me what was going on:

```bash
$ pgrep -P $SERVER_PID
2254
$ cat /proc/$SERVER_PID/status | grep -E 'Name|VmRSS|Threads'
Name:   bash
VmRSS:  2144 kB
Threads:  1
$ cat /proc/2254/status | grep -E 'Name|VmRSS|Threads'
Name:   llama-server
VmRSS:  523804 kB
Threads:  52
```

`$!` in the shell command had captured the PID of the bash process that ran the redirection, not the `llama-server` process it ultimately spawned. The two are different. The wrapper sits at 2 MiB, single-threaded, idle — completely truthful information about a completely uninteresting process. The actual workload is in its child.

The bug, then, isn't in inferscope. inferscope had faithfully sampled the process it was told to sample. The bug is in the assumption that the PID you have in your hand is the PID doing your work.

This pattern is everywhere. `gunicorn` arbiter spawns worker processes. `uvicorn` master spawns workers. `vllm.entrypoints.openai.api_server` loads the engine in a child. Anything that uses `fork()` as a startup pattern — which is most production server software on Linux — has this shape. The wrapper-PID failure isn't a quirk of `llama.cpp`. It's the default architecture of the software inferscope exists to profile.

## Three responses, in order of cost

I thought about three ways to handle this, and ended up doing all of them in sequence over the next four days.

**Documentation.** First commit was a runbook change: when you start an inference server with shell backgrounding, capture the worker PID explicitly with `pgrep -P` or by reading `/proc/<parent>/task/<parent>/children`. This costs zero engineering effort and offloads the entire problem onto the user. It's the right minimum response — a tool that surprises its user should at least say "here's what to watch out for." But it doesn't fix anything; it just makes the failure mode legible.

**Runtime warning.** Second commit added an idle-PID heuristic to the CLI. If across every sample we collected the RSS stays below 10 MiB **and** there's exactly one thread **and** CPU jiffies are zero, the tool prints a stderr warning suggesting the PID may be a wrapper. The triple condition is the conservative version — each criterion alone has false positives (a small daemon, a single-threaded process, a momentarily idle one), but together they describe a process that is, with very high confidence, not the workload you meant to measure. The warning is informational; it doesn't fail the run or change the report. This puts the diagnosis in the user's face at the moment they're looking at suspicious numbers.

**Aggregation.** Third commit, which became inferscope v0.2.1, is the real fix: when invoked with `--include-descendants`, the sampler reads `/proc/<pid>/task/<pid>/children`, samples each child's `/proc/<child>/{status,stat}` alongside the parent, and sums the numeric fields. The user can keep passing the wrapper PID; the tool does the right thing.

The three responses are not alternatives. The documentation makes the user aware. The warning catches the case the documentation didn't reach. The aggregation lets the user fix it without rethinking their shell pattern. They compose.

## What the aggregation actually does

I wrote up the design as [ADR-006](https://github.com/MicheleCampi/inferscope/blob/main/docs/adr/006-process-tree-aggregation.md). The decisions worth surfacing here, in compressed form:

**Direct children only.** The sampler reads one level of the process tree. It does not recurse into grandchildren. For inference engines this is enough — `llama-server` forks one worker, `vllm` spawns workers from a master at one level, `uvicorn` and `gunicorn` follow the same shape. Recursive walking would be unbounded, would need cycle detection, and would not buy anything for the v0.2.1 audience. If a production deployment surfaces a multi-level case, that's a v0.3 conversation.

**Sequential, not parallel.** Reading `/proc` for a single PID is microseconds. For N ≤ 10 children, sequential reads complete in tens of microseconds total. Parallelising via `tokio::join!` would add task-scheduling overhead larger than the I/O it parallelises. The loop stays flat and trivially testable.

**Saturating arithmetic.** Every sum uses `saturating_add`. The `u64` resource fields (RSS bytes, CPU jiffies) cannot realistically overflow on a Linux host. The `u32` thread count, in theory, can — and using `checked_add` to propagate an error for a corner case the user cannot act on would be worse ergonomics than just saturating.

**Failure-tiered behaviour.** Three failure paths get three different responses. If the parent's `/proc/<pid>/status` is unreadable, the whole sample fails and propagates an error — the parent PID is the user's anchor and meaning depends on it. If the `children` file is unreadable, the sampler falls back to parent-only — rare kernel/namespace configurations can hide it, and a parent-only sample is still useful. If a specific child is unreadable, it is silently skipped — children exit between discovery and read, all the time, and this is the normal case.

The integration test that exercises the path is simple: spawn `bash -c "sleep 30 & wait"`, capture the bash PID, sample it twice — once with `sample_once`, once with `sample_once_aggregated`. The load-bearing assertion is `aggregated.thread_count > parent_only.thread_count`. bash on its own is single-threaded; if the aggregated count is strictly higher, the only way that can happen is if the sampler actually walked the children file and read the child's status. There is no other code path that produces that result.

## Re-validation on H100

A week later I rented an H100 SXM on RunPod (~$3.29/hr) and reproduced the entire pattern. Same `llama.cpp` build, same shell pattern with `$!`, same wrapper-and-worker shape. This time the model was Qwen 2.5 7B Q4 — fourteen times larger than the 0.5B, the size H100's 80 GB of HBM3 invites you to try.

The wrapper-PID failure showed up identically. `cat /proc/$WRAPPER/status` reported `bash`, 2.25 MiB RSS, 1 thread. `cat /proc/$WORKER/status` reported `llama-server`, 720 MiB RSS, 228 threads. The undercount magnitude grew with the workload: on H100 + 7B, an inferscope run pointed at the wrapper sees 0.3% of the real RSS and 0.4% of the real thread count.

I ran three back-to-back runs:

| | A1 (wrong PID, no fix) | A2 (correct PID, baseline) | B1 (wrong PID + `--include-descendants`) |
|---|---|---|---|
| stderr warning | fires | none | **none** |
| TTFT | 54.4 ms | 39.7 ms | 25.5 ms |
| tokens/s | 227.4 | 230.4 | 228.7 |
| **RSS peak** | **2.25 MiB** ❌ | 720 MiB ✓ | **722 MiB ✓** |
| **Threads peak** | **1** ❌ | 228 ✓ | **229 ✓** |
| **CPU mean** | **0%** ❌ | 90.72% ✓ | **90.77% ✓** |
| VRAM peak | 5.56 GB | 5.56 GB | 5.56 GB |
| SM mean | 40% | 46% | 48% |
| Power mean | 178 W | 170 W | 177 W |

B1 reproduces A2 within 0.3% on RSS, +1 thread (which is bash itself), 0.05 percentage points on CPU. The fix does not impersonate the correct PID — it sums honestly. The +1 thread is the bash wrapper, which is the right accounting. And because the aggregated sample is no longer "idle" (RSS well above 10 MiB, threads above 1, CPU above zero), the v0.2.0 idle-PID warning stops firing on its own — the fix is transparent to the user.

The timing section drifts across the three runs in the way you'd expect from cold-vs-warm CUDA caches between back-to-back invocations: 54 → 40 → 25 ms TTFT as the GPU gets warmer. The GPU section is invariant across the three runs because NVML reads the device directly — that path was never broken; only the `/proc` path was.

## What the H100 actually does

This is the part the fix made possible. With aggregation working, the resource numbers are trustworthy, and the question becomes: what is the H100 actually doing while it serves Qwen 7B Q4?

The honest answer is "not very much." 5.6 GB of 80 GB VRAM, 6.5%. SM utilisation mean 48%, peak 90% — the chip spikes during computation, idles during streaming. Power mean 170 W on a 700 W TDP, 24%. The chip is loafing.

I then loaded Qwen 2.5 32B Q4 (~18.7 GB) on the same hardware and ran the same prompt:

| | H100 + Qwen 7B Q4 | H100 + Qwen 32B Q4 |
|---|---|---|
| Throughput | 230 tokens/s | 69 tokens/s |
| TTFT | 40 ms | 88 ms |
| Inter-token p50 | 4.3 ms | 14.4 ms |
| VRAM used | 5.6 GB (6.5%) | **21.4 GB (25.1%)** |
| SM mean | 48% | **88%** |
| Power mean | 170 W | **439 W** |
| Power / TDP | 24% | **63%** |
| CPU mean | 90.72% | 98.81% |

Same hardware. Same `llama.cpp`. Same inferscope. The 32B run uses 1.8× more SM, 2.6× more power, 3.8× more VRAM. The chip moves from loafing to genuinely compute-bound. Throughput drops 70% — each token costs more cycles, that's expected — but the resource utilisation tells a story the throughput alone doesn't.

For comparison, the L4 run with Qwen 0.5B Q4 was using 51% of its power envelope. The L4 was getting closer to what you paid for than the H100 was, on the smaller model. If your workload is 7B-class, the H100 is overprovisioned: you are paying $3.29/hr to use ~25% of a chip when an L4 at $0.45/hr would use ~50% of a chip and serve roughly the same model. The economics on that swap are easy. If your workload is 32B-class, the H100 is finally earning its hourly rate.

There is nothing surprising about any of this if you've thought about it. What is surprising is how rare it is to actually measure it. Most people I've talked to who run inference on rented H100s do not know what fraction of the chip they're using on a sustained basis. They look at `nvidia-smi` once during a run and see "yes the GPU is doing something" and conclude the hardware is appropriate. The hardware might not be appropriate. The profiler is what tells you.

## What the hardware taught me about the profiler

I built inferscope to learn things about hardware. The first thing I learned was a bug in the profiler. The second was that the hardware I was paying for was mostly not the hardware I was using.

Both lessons came from running the tool on a GPU I hadn't run it on before. The bug existed in v0.1 and v0.2.0; it just hadn't surfaced because every previous validation pointed at a PID that happened to be the actual workload. The H100 didn't introduce the bug — it forced me to use the tool in a way (longer, bigger model, slower iteration cycle on rented hardware) where the wrong numbers were too obvious to ignore. The under-utilisation insight is structurally similar: it was true on the L4 too in some sense, but it took a 7B model on an 80 GB GPU to make the ratio visceral.

If there is a generalisable point here, it is that profilers exist to make the obvious legible. The wrapper PID was obvious — anyone who knows `pgrep` could have found it — but inferscope was the thing that made the obviousness uncomfortable enough to fix. The H100 was overprovisioned for a 7B model — anyone who looks at VRAM percentages could have figured it out — but having the SM and power numbers in the same report as the per-token timing is what turned "could have figured it out" into "did figure it out."

inferscope v0.2.1 is on GitHub, Apache-licensed, cross-hardware validated on RTX L4 and H100. The next release is multi-GPU support; production inference is rarely single-device, and a profiler that can't see tensor-parallel workloads is solving half the problem. If you want to try it against your own inference server, the [README](https://github.com/MicheleCampi/inferscope) walks through the setup. If you've found similar wrapper-PID class bugs in your own observability stack, I'd be interested to hear about it.
