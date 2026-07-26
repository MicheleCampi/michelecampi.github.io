---
layout: post
title: "CUDA graphs always speed the kernel. They don't always speed the server."
subtitle: "Measuring the enforce_eager trade-off in vLLM across two model sizes and two load regimes — where the sign flips, and why."
date: 2026-07-26
categories: observability systems-engineering llm-inference
tags: rust llm profiling observability gpu h100 inferscope vllm cuda-graphs
---
In an earlier post I traced vLLM's cold start at the kernel level with eBPF, and one line stood out: most of the ~18 seconds before a model serves its first token is GPU warmup rather than disk I/O. That post ran with `enforce_eager=True` to keep the trace readable, so it could not measure graphs at all; it closed by naming CUDA graph capture as one of the plausible levers on that warmup. I flagged it and moved on. CUDA graphs are supposed to be free money at inference time — capture the decode step once, replay it without per-kernel launch overhead — so paying for them at startup seemed like an obvious trade worth making.

This post is me going back to actually measure that trade. Not "are CUDA graphs faster" — they are, that question is settled — but: how much do they cost at startup, how much do they pay back at steady state, after how many requests does the payback clear the cost, and does any of that change with model size and load?

The answer is more interesting than I expected. CUDA graphs speed up per-token decode in every single configuration I tested. And yet, on the larger model under sustained load, turning them on made the server *slower* end to end. The benefit is real and the regression is real, in the same run, measured the same way. The rest of this post is about how both can be true at once.

## The setup

One variable, isolated: `enforce_eager`. With `enforce_eager=False` (vLLM's default) CUDA graphs are captured and used; with `enforce_eager=True` the engine runs eager, no graphs. Everything else held constant.

- **Engine:** vLLM 0.23.0, single H100 80GB PCIe (Lambda), CUDA 13 / driver 580. The energy re-run in "The third axis" is on this same card.
- **Models:** Qwen2.5-7B-Instruct and Qwen2.5-32B-Instruct, both bf16.
- **Shared context budget:** `--max-model-len 4096` on *both* models. This matters for a clean cross-model comparison — left at the 32K default, the 32B's KV cache on one H100 allows barely ~1x concurrency, which would make the sustained test measure cache thrashing instead of CUDA graphs. Fixing the budget at 4096 (the benchmark uses 1024 in + 128 out, so it's ample) keeps the only cross-model difference the model size itself.
- **Load profiles:** *sporadic* (50 prompts, concurrency 1) and *sustained* (500 prompts, concurrency 64).
- **Two measurement planes:** inferscope captures cold start and per-device GPU (NVML) during a single probe request; `vllm bench serve` drives the steady-state load with a fixed random 1024/128 dataset and reports throughput, TTFT, TPOT.
- **Matrix:** 2 models × 2 conditions × 2 profiles × 5 reps = 40 runs, plus cold-start capture. Idempotent orchestrator, warmup excluded, version recorded at runtime.

All 40 result files and the analysis script are in the repo; every number below is regenerable with `deep_analysis.py`.

## The cost: cold start

Turning graphs on costs startup time, and the cost scales with the model. Measured from process start to first token, pooling both load profiles (cold start is measured before the bench, so the profile does not touch it) and dropping one replicate contaminated by a cold compilation cache:

| model | cold start ON | cold start OFF | extra startup for ON |
|---|---|---|---|
| 7B  | 37 s (n=9)  | 30 s (n=10) | **+7.0 s** |
| 32B | 56 s (n=10) | 40 s (n=10) | **+15.9 s** |

But calling that "the cost of graph capture" would be wrong, and the server log says so. `--enforce-eager` does not only skip capture: it disables the compiled path as well, so the OFF arm never runs `torch.compile` either. The engine logs both phases separately, and they split the delta differently on the two models:

| model | Δ startup | graph capture | `torch.compile` | accounted |
|---|---|---|---|---|
| 7B  | 6.15 s (engine init) | 3.0 s (49%) | 2.9 s (47%) | 96% |
| 32B | 14.95 s (engine init) | 9.1 s (61%) | 5.2 s (35%) | 96% |

So on the 7B, capture and compilation cost about the same; on the 32B capture dominates but still leaves a third of the bill to the compiler. That is the answer to the lever the eBPF post pointed at: graphs are a real part of the warmup bill, but on a small model they are only about half of what turning them off actually saves.

Two practical notes on this measurement. Capture happens *inside* engine init and finishes before the server accepts its first request — 3 seconds and 0.58 GiB on the 7B, 9 to 10 seconds and 1.01 GiB on the 32B. Nothing about CUDA graphs is paid at request time. And the compilation figures above are warm-cache: the first start of each campaign writes vLLM's AOT compile cache and pays 27 s on the 32B instead of 5.2 s, which is why one replicate is excluded from the table rather than averaged into it.


## The payback, and the break-even

For a long-running server, +7s of startup is nothing. The question is whether the steady-state gain ever repays it, and how fast — and the answer depends on the load regime, not just on the model.

Under *sustained* load the gain is aggregate throughput. On the 7B, graphs lift output throughput from ~2007 to ~2102 tok/s (+4.75%), which amortises to ~2.9 ms per request at 128 output tokens. Divide the 7.0s startup penalty by that saving and the 7B repays its graphs after roughly **2,450 requests** — minutes of steady traffic.

Under *sporadic* load, at concurrency 1, the gain is not aggregate throughput but per-request latency, and per request it is two orders of magnitude larger. The 7B decodes at 9.11 ms/token with graphs against 9.93 without: about 105 ms saved on a 128-token response. The same 7.0s penalty now clears in roughly **70 requests**.

| model | regime | saving per request | startup penalty | break-even |
|---|---|---|---|---|
| 7B  | sporadic (conc. 1)   | ~105 ms  | 7.0 s  | **~70 requests** |
| 7B  | sustained (conc. 64) | ~2.9 ms  | 7.0 s  | **~2,450 requests** |
| 32B | sporadic (conc. 1)   | ~171 ms  | 15.9 s | **~90 requests** |
| 32B | sustained (conc. 64) | negative | 15.9 s | **never** |

These are order-of-magnitude figures. The cold-start measurement is quantised to a ~2s polling grid (see Reproducibility), so "~70 requests" carries something like ±10.

This inverts the intuition I started with. A scale-to-zero replica that spins up, serves a handful of requests and dies looks like the case where graphs are least affordable — and it is the case where they repay fastest, because at concurrency 1 every request collects the full per-token win instead of amortising it across a saturated batch. The deployment that should think twice is the opposite one: the large model pinned permanently under load, where the last row refuses to produce a break-even at all.

That row is the one worth pulling on.


## The finding: the sign flips

Here is the throughput delta (ON relative to OFF) across all four model×load cells, ordered by how far each pushes the engine past its measured KV-cache concurrency:

| model | profile | queue overload | Δ throughput (ON vs OFF) |
|---|---|---|---|
| 7B  | sporadic  | 0.0x | **+8.51%** |
| 32B | sporadic  | 0.1x | **+3.21%** |
| 7B  | sustained | 2.0x | **+4.75%** |
| 32B | sustained | 7.6x | **−5.15%** |

On the 32B under sustained load, turning CUDA graphs *on* cost 5% of throughput. Not within noise — the five-rep distributions are cleanly separated, ON in [315.8, 318.0], OFF in [332.4, 335.3], no overlap. ON also paid 15.9s more startup to get there. On that workload, graphs are pure loss.

This is the opposite of the 7B, where ON wins in every profile. Same engine, same code path, same flag — the trade-off changes sign between the two model sizes under load.

## Making sure it's real

A surprising result is a liability until you've tried to kill it. I went after the alternative explanations one at a time.

**Is it noise?** No. Five reps per cell, ON and OFF distributions separated with no overlap in all four cells, standard deviations around 1%.

**Is it measurement drift between reps?** No. The five values per cell oscillate, they don't trend — no cache-warming or state contamination across reps.

**Is the slower run doing less work?** No. Both ON and OFF complete all 500 requests and emit exactly 64,000 output tokens. Same work. What differs is wall-clock: ON takes 202.0s, OFF 191.6s — ON is 5.4% slower to clear identical work.

**Is it preemption / KV recompute under pressure?** This was my first guess — graphs constrain shapes, maybe the engine preempts and recomputes more under saturation. The data says no: inter-token latency tracks TPOT almost exactly in every cell (ITL ≈ TPOT to the second decimal). Preemption with recompute would blow ITL far past TPOT and explode the p99 tail. It doesn't. There are no preemption events in the server logs either.

So the regression is real, it's the same work, and it isn't preemption. Where does the time go?

## The decisive measurement: decode is faster, the server is slower

The key number is per-token decode latency (TPOT), and it points the *other* way from throughput:

| model | profile | TPOT ON | TPOT OFF | ON decode |
|---|---|---|---|---|
| 7B  | sporadic  | 9.11  | 9.93  | +8.27% faster |
| 7B  | sustained | 19.70 | 21.28 | +7.43% faster |
| 32B | sporadic  | 38.00 | 39.34 | +3.40% faster |
| 32B | sustained | 78.21 | 82.74 | **+5.48% faster** |

CUDA graphs make per-token decode faster in **all four comparisons**, the 32B-sustained cell included — the very cell where end-to-end throughput is 5% *worse*. Decode speeds up, the server slows down, in the same run. That's not a contradiction; it's a localization. The graphs do their job at the kernel — per-token compute is faster everywhere. The loss is entirely above the kernel, in scheduling and batching.

inferscope also samples the GPU per-device, but in the original 40-run matrix it only sampled during the cold-start probe — a ~0.65s near-idle window, not the sustained load. The power figures from that window sit near 150W, which is an almost-idle card, not the bench; I don't rely on them. To see the GPU under actual load I re-ran the sustained cells with inferscope attached for the whole bench window and NVML's energy counter active. That measurement is the next section, and it turns this hand-wave into a number.

## The third axis: energy

Throughput and latency are two axes; the one that pays the cloud bill is the third — energy per token. I re-ran the sustained cells with inferscope sampling the full bench window through NVML's hardware energy counter (`nvmlDeviceGetTotalEnergyConsumption`, an end-minus-start delta over the load, not an integral of power samples), three reps per cell.

One honesty note, and it's the opposite of a caveat: this energy run is on the **same H100 PCIe** as the original throughput and TPOT matrix — same card, same ~350W nominal envelope (transient boost peaks reach 381W), same vLLM 0.23.0 build. So this isn't a second measurement on a different machine that happens to agree; it's a fully independent instrument — NVML's hardware energy counter — pointed at the same hardware in the same regime. Throughput is a scheduling-layer metric; energy-per-token is a hardware register. They share nothing in the measurement path. When they land on the same sign, that's two witnesses, not one.

Token denominator is the bench's own `total_output_tokens` (64000, capped identically on both models — no early-EOS), so 7B-vs-32B is a clean cross-model comparison.

| model | tok/J ON | tok/J OFF | energy Δ |
|---|---|---|---|
| 7B  | 4.660 | 4.646 | +0.3% |
| 32B | 0.790 | 0.804 | **−1.8%** |

Two stories, same split as the throughput table:

- **7B: no measurable energy effect.** The +0.3% tok/J is inside run-to-run scatter, and the 7B's short window makes even that figure soft (see the note below). What the measurement supports is a bound, not a sign: on the small model, graphs buy a large throughput win without any energy cost per token that this instrument can resolve.
- **32B under overload: graphs lose on *both* axes.** The sections above already showed graphs-on losing throughput on this cell; the energy counter shows the same direction — 1.8% more joules per output token. Not a trade where you spend efficiency to buy speed; you lose both.

That second row is the physical confirmation the two-factor model below predicts. The model says the padding loss "burns power without producing tokens"; here the counter measures it. And it localizes *how*: mean board power rises 199.3W→203.0W (+1.9%) on the saturated 32B. That rise is not an artefact of the slower arm occupying more of a fixed window — within the ON arm alone, the replicate with the *most* active time draws the *least* power (active 0.763 at 202.4W against 0.738 at 203.5W), which is the opposite of what window dilution would produce. Where the two-factor model says padding burns power without producing tokens, the counter agrees, and says it burns it as watts.

(All twelve runs reported `energy_source: counter` — the NVML hardware register, never the integrated-power fallback. One caveat on the absolute numbers: the sampling window is fixed per model (400s on the 32B, 60s on the 7B) and brackets the bench, so it also covers a head of roughly 15s while `vllm bench serve` loads and tokenises its dataset, plus a tail after the run drains. On the 32B that overhead is small — token generation spans ~290s of the 400s window on both arms. On the 7B it is not: generation spans as little as 36s of 60s, so up to 40% of the integrated energy is collected while no tokens are being produced, and the arms differ in how much. I therefore read the 7B row as *no measurable effect* rather than as a signed +0.3%; the 32B row, where the window is 95% useful and the effect is six times larger, is the one that carries weight.)

## A two-factor model

One more cut explains the whole table. Look again at the decode-improvement column above: the per-token gain from graphs is consistently *larger on the 7B* (~7.9% average) than on the 32B (~4.4%). On a small model, per-kernel launch overhead — exactly what CUDA graphs eliminate — is a bigger fraction of each decode step, so removing it helps more. On the 32B the kernels are large and GPU-bound, launch overhead is already diluted, the gain is smaller.

That gives two opposing forces:

1. **Base gain (compute), shrinks with model size.** The kernel-level win from graphs. Large on 7B, modest on 32B. Stable across load.
2. **Padding loss (scheduling), grows with queue overload.** vLLM captures graphs at a discrete set of batch shapes. Under saturation, when the queue is full and batches are large and variable, work gets padded to the nearest captured shape — burning the power inferscope sees, without producing tokens. Small when the queue is shallow, large when it's deep.

The sign of the throughput trade is just which force wins. On the 7B the base gain is large enough to dominate even at 2.0x overload (ON still wins sustained). On the 32B the base gain is already modest and the overload is severe (7.6x, forced by the limited KV cache) — so the padding loss takes over and the sign flips.

I want to be precise about the epistemic status here. The base-gain factor is measured directly (TPOT, isolated from scheduling). The overload trend is measured directly (the four-cell ordering). The *padding-to-captured-shape* mechanism is the best-supported explanation of where the scheduling loss comes from — it's consistent with every measurement — but I have not yet instrumented vLLM's scheduler to watch the padding happen. That's the natural next step, and the subject of a follow-up: add a metric at the batching layer, reproduce the saturated 32B, and turn the hypothesis into a proof.

## Practical guidance

Concrete takeaways for tuning `enforce_eager`:

- **Default (graphs on) is right for the common case:** small-to-mid models, long-lived replicas, traffic that doesn't sit permanently past KV-cache saturation. The decode win is real and the startup cost amortizes fast.
- **Short-lived replicas are not the problem I expected.** At concurrency 1 the per-request decode win is large (~105 ms on the 7B), so a scale-to-zero replica repays its +7s of startup in roughly 70 requests, and the 32B its +16s in about 90. Graphs are the right default for serverless too — unless a spin-up serves only a few dozen requests before dying.
- **Reconsider graphs for large models pinned under saturation.** If a big model runs permanently past its KV-cache concurrency on a single device, captured-shape padding can make graphs a net throughput loss despite faster per-token decode. Measure your own workload — the sign depends on the overload factor, not just the model.

The headline a lot of tuning guides give — "enable CUDA graphs, they're faster" — is true at the kernel and incomplete at the server. The kernel is always faster. Whether the server is faster depends on how big the model is and how hard you're pushing the queue.

## Reproducibility

The 40 throughput result files, the 12 energy JSON in `results-energy-tight/` (the published energy set; `results-energy/` is an earlier campaign kept for provenance and superseded), the orchestrator, `deep_analysis.py` (regenerates the throughput evidence chain — aggregates, raw distributions, noise and preemption checks, two-factor table, break-even) and `aggregate_energy.py` (re-derives every tok/J and divergence number in "The third axis", with guard-rails that fail loudly if the token denominator or the per-phase energy split don't reconcile) are all in the experiment repo: [MicheleCampi/cuda-graphs-experiment](https://github.com/MicheleCampi/cuda-graphs-experiment/tree/v1.0.0), pinned at tag `v1.0.0` so the tree behind these numbers stays fixed even as `main` moves. `validation-results/PROVENANCE.md` pins the exact hardware, driver, and CUDA build, since the JSON carry no hardware tag. Clone it, run the scripts, and you get the numbers in this post. Cold-start and per-device GPU figures come from inferscope; throughput from `vllm bench serve`; energy from NVML's hardware counter. Three measurement planes, one clock, every claim traceable to a file. One limit of the first plane worth stating: readiness is polled on a ~2s grid, so cold-start values are quantised in ~2s steps and the per-cell spreads reflect that grid rather than physical variance. The grid also delays detection by about a second on average, which cancels in the ON−OFF delta but inflates the absolute figures. The deltas reported here (7s and 16s) are several times the step; nothing in this post rests on resolving cold start more finely than that.
