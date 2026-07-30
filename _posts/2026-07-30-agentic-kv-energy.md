---
layout: post
title: "KV-cache reuse is an energy lever. Per-token attribution can't see it."
subtitle: "+69.2% tokens/joule from cold prompts to high prefix reuse on an H100, measured across an 18-cell matrix — and why the token-share apportionment of that same energy stays flat at 0.996 the whole way."
date: 2026-07-30
categories: observability systems-engineering llm-inference
tags: rust llm profiling observability gpu h100 inferscope vllm kv-cache energy agentic
---
Prefix caching is one of the least controversial optimisations in LLM serving. An agentic loop sends the same system prompt and the same growing conversation history on every turn, the engine keeps the attention state for tokens it has already seen, and the prefill it would otherwise redo becomes a cache lookup. The case for it is always made in latency and throughput: time to first token drops, requests per second go up.

Energy is not usually part of that case. It should be, because the mechanism is not "the same work, faster" — it is *less work*. A cache hit does not accelerate the attention computation for those tokens; it skips it. Skipped compute is joules that never leave the wall, and on a fleet where GPUs are the line item that matters, that is the number a platform team can act on.

So: how much is prefix reuse actually worth in joules, on a workload shaped like a real agentic loop, with cache hit-rate as the only thing that varies?

This post answers that with an 18-cell measurement matrix on a single H100 SXM5 — three hit-rate regimes, two traffic conditions, three repetitions each — and then reports a second result I was not looking for. The energy gradient is large and clean. But the two ways of attributing that energy to prefill and decode disagree, the disagreement grows monotonically with hit-rate, and it grows because one of the two is structurally blind to cache reuse. The first result is the headline. The second is the one I would want a platform engineer to read.

## What is being varied, and what is being held still

The hit-rate is not a flag. There is no `--hit-rate=0.9` to set, and a benchmark that pretended otherwise would be measuring its own configuration rather than an engine. Cache reuse is a property of the *shape of the traffic*: how much of each request's prompt the engine has seen before. So the three regimes are induced by generating traffic with different amounts of shared prefix, and the realised hit-rate is then read back from the engine's own counters as a measured coordinate.

| Regime | Traffic shape | Realised hit-rate (measured) |
|---|---|---|
| H0 | seed-disjoint prompts, no shared prefix | 0.0000 exact, all 6 cells |
| H1 | partial history sharing | 0.449 – 0.548 |
| H2 | shared-prefix heavy | 0.931 – 0.934 |

Each regime runs under two conditions — `nominal`, and a `failure` condition that injects tool-call failures and the retry traffic they produce, because an agentic loop that never fails is not an agentic loop — and each combination runs three times. Eighteen cells, zero aborts, zero excluded cells.

Four things are deliberately held still, and all four matter when reading the numbers:

**CUDA graphs are off.** Every run uses `--enforce-eager`, hardcoded in the orchestrator as a matrix invariant rather than passed per-run. [An earlier post measured what graphs cost and pay back](/observability/systems-engineering/llm-inference/2026/07/26/cuda-graphs-tradeoff.html) and found the throughput sign can invert under saturation; that is a second lever with its own trade-offs, and leaving it enabled here would mean two causes on one curve. The knob is the hit-rate. Absolute tok/J figures below are therefore eager-mode figures, and the gradient is what transfers, not the level.

**One engine, one model, one card.** vLLM 0.23.0, Qwen2.5-32B-Instruct, a single H100 80GB SXM5, 9.35 GiB of KV cache available after weights — a value confirmed identical on a third node, which is how I know the KV allocation is a property of the configuration and not of the machine I happened to rent.

**Generation is capped.** Eight requests per cell at `--max-tokens 128`, so `generation_tokens_delta` lands at 1024 in seventeen of eighteen cells (one cell hit an early EOS at 884). Prompt tokens run 217–280× that. The workload is prefill-dominated *by construction* — which is the point, since that is the shape of an agentic loop with long shared context and short tool-call outputs, but it also means nothing here speaks to decode-heavy serving.

**The first request after engine start is thrown away.** The system prefix is 14,785 tokens and it misses exactly once per engine start, which is enough to depress whichever cell runs first. So a discarded warm-up cell runs before the matrix, sending the bare prefix twice, marked `"excluded from all statistics"` in its own manifest and deliberately not resume-safe: its purpose is tied to *this* engine start, so a resumed campaign must re-run it rather than skip it.

## Result 1: +69.2% tokens per joule

The metric is `(prompt_tokens_delta + generation_tokens_delta) / window energy`. Tokens come from the engine's own Prometheus counters, differenced over the measurement window. Energy comes from NVML's cumulative energy counter, differenced over the same window — a counter read, not an integration of power samples, on all 18 cells. The window is a fixed 120 seconds, sampled at 50 ms for 2400 samples per cell.

| Regime | Realised hit-rate (nominal cells) | tok/J (mean ± std, n=3) | vs H0 |
|---|---|---|---|
| H0 | 0.0000 exact | 5.692 ± 0.147 | — |
| H1 | 0.449 – 0.515 | 6.923 ± 0.091 | +21.6% |
| H2 | 0.933 | 9.628 ± 0.019 | **+69.2%** |

Monotone in hit-rate, per-regime standard deviation under 3% of the mean everywhere, and tightest at H2 where the hit-rate itself is most stable (0.019 on 9.628, two tenths of a percent). Gross energy over the fixed window falls from 42.8 kJ at H0 to 26.5 kJ at H2 (means over six cells; H2 spans just 26.4–26.5 kJ).

The `failure` condition tracks nominal closely — 5.449 ± 0.103, 6.846 ± 0.046, 9.610 ± 0.029 — so the gradient survives failure and retry traffic essentially intact. That is worth stating plainly because it was the result most likely to break: retries inflate unique token mass, which dilutes hit-rate, which should have flattened the curve. And the penalty itself shrinks as the cache warms: −4.3% at H0, −1.1% at H1, −0.2% at H2. Extra unique token mass costs you where prefill is expensive and costs you almost nothing where it has already been skipped.

### Why this is a floor and not a point estimate

The window is fixed at 120 seconds; the work is not. The generator finishes and the GPU goes idle for the remainder, and that idle tail is *longest exactly where tok/J is highest*, because higher reuse means less prefill means the same work finishes sooner. Window-based tok/J therefore penalises the good regimes more than the bad ones.

I can now put a number on that tail rather than estimating it, because the per-second phase timeline in each report carries the engine's cumulative counters, and the sample at which they stop advancing is the instant the engine went quiet:

| Regime | Active time (engine-side, n=6) | Idle share of the 120 s window |
|---|---|---|
| H0 | 62.8 s ± 2.1 | 47.6% |
| H1 | 50.2 s ± 1.8 | 58.2% |
| H2 | 36.2 s ± 0.4 | 69.9% |

Those active times agree with the generator's own wall-clock to within 0.4–0.9 s on all eighteen cells, which is a cross-check from the opposite side of the connection: client-side wall time and engine-side counter advance independently say the same thing about when the work ended.

So **+69.2% is a lower bound**. An active-window figure would widen the gradient, not shrink it. I am not publishing an active-window number, because doing it honestly needs per-sample energy rather than a window counter delta, and this campaign persisted aggregate power statistics only — an instrumentation gap tracked as backlog, not something I can patch after the fact. The published metric is the one that is fully backed by evidence and requires no idle-power estimation.

## The control that makes the gradient readable

A 69% swing in efficiency is only interesting if you know what moved. The engine's phase counters answer that directly, because they separate time spent in prefill from time spent in decode, and the two behave completely differently across the matrix:

| Regime | Prefill time (n=6) | Decode time (n=6) |
|---|---|---|
| H0 | 30.66 s | 27.13 s |
| H1 | 17.09 s | 27.89 s |
| H2 | 2.85 s | 27.91 s |

Prefill collapses by 91%. Decode stays put: 27.9 s at both H1 and H2, with standard deviations of 0.09 s and 0.01 s across six cells each. H0's decode mean sits slightly lower and much wider (27.13 s ± 1.50) for a reason visible in the per-cell data — one H0 cell hit an early EOS and generated 884 tokens instead of 1024, so it decoded less. Excluding nothing and explaining it is the honest version: the cap holds generation constant, and where it did not, the deviation is accounted for.

That is what a controlled experiment is supposed to look like, and it is the strongest internal evidence in the campaign: the cache knob acts on prefill and only on prefill, the generation cap held generation constant as designed, and there is no second mechanism hiding inside the energy gradient.

It also explains the shape of the tok/J curve without any appeal to theory. The regimes do not differ in how fast they generate; they differ in how much prompt they have to compute before generating. At H2 the engine reuses cached state for 93% of the prompt tokens it is asked about, and the prefill time that would have cost drops by 91%. The joules follow the compute that never happened.

## Result 2: the accounting disagrees with itself, and it disagrees more the better the cache works

NVML gives one energy number for the device over the window. It does not know what the device was doing. So if you want to say how much of that energy went to prefill and how much to decode, you have to apportion it, and there is no measurement that does it for you — you pick a basis.

The profiler I use computes two, deliberately, and treats neither as the answer:

    share_prefill_time = prefill_ns / (prefill_ns + decode_ns)
    share_prefill_tok  = prompt_tokens / (prompt_tokens + generation_tokens)

Time-share assumes energy tracks time-in-phase. Token-share assumes energy tracks tokens processed. Both are wrong in known ways — prefill is compute-bound and draws more power per second than memory-bound decode — so the design decision was to publish the *difference between them* as the load-bearing signal rather than pick a winner. Across this matrix, that difference does something worth looking at:

| Regime | `share_prefill_time` | `share_prefill_tok` | divergence (n=6) |
|---|---|---|---|
| H0 | 0.505 – 0.572 | 0.9954 – 0.9964 | −0.465 ± 0.023 |
| H1 | 0.355 – 0.414 | 0.9956 – 0.9959 | −0.616 ± 0.021 |
| H2 | 0.092 – 0.094 | 0.9960 | −0.903 ± 0.0009 |

The three bands do not overlap, and not marginally: the closest approach is between H0 and H1, where H0's least negative cell sits at −0.424 and H1's least negative at −0.582, a gap of 0.16 with per-cell spreads of 0.07 and 0.06. Between H1 and H2 the gap is 0.26.

And the movement is entirely on one side. `share_prefill_tok` is flat at 0.996 across all eighteen cells, with a total spread of 0.001 between the most and least cache-friendly cell in the matrix. `share_prefill_time` falls from 0.52 to 0.09. The divergence grows because time-share tracks the cache and token-share does not move at all.

### Why token-share cannot see the cache

Look at what goes into it. `prompt_tokens` counts the tokens in the requests the engine was asked to process. Nothing in that counter distinguishes a token the engine computed attention for from a token it retrieved from cache — both are prompt tokens, both increment the counter identically. The denominator has no term that responds to reuse. So as the cache goes from cold to 93% warm and the actual prefill work drops by an order of magnitude, token-share keeps assigning 99.6% of the device's energy to prefill, because 99.6% of the tokens are still prompt tokens.

Time-share, whatever else is wrong with it, is at least measuring something that moved. Prefill went from 30.66 s to 2.85 s and the apportionment followed.

This is what the divergence is for. Neither number is the truth — one is anchored to a quantity that responds to the thing you changed, the other is anchored to a quantity that structurally cannot. Publishing the gap between them makes that visible instead of burying it inside whichever basis you happened to pick.

### The part I have not measured

There is an obvious analogy here and I want to name it as an analogy rather than smuggle it in as a finding. Inference is commonly billed and budgeted per token, usually with input tokens priced below output tokens. That pricing carries the same structural property as token-share apportionment: a cached prompt token and a computed prompt token are the same line item, even when one costs an order of magnitude less to serve.

I have not measured any billing system, and the commercial providers have plainly already priced this in: cache-read rates at roughly a tenth of standard input pricing are now the norm across the major APIs, which is the market's way of saying the gap is real and large. That is an argument for the mechanism, not against it. What I have measured is that on this workload, the energy actually consumed per unit of shared prefix falls by a factor that per-token counting does not register. If you run your own fleet and you allocate GPU cost to teams or workloads on a per-token basis, that is a gap worth checking on your own numbers, because the mechanism producing it is not specific to my setup — it is arithmetic about what a token counter counts.

## What this does not show

**One model, one card, one engine.** Qwen2.5-32B on a single H100 SXM5 under vLLM 0.23.0. The mechanism is not model-specific — skipped attention is skipped attention — but the magnitude is a property of this configuration, and 9.35 GiB of KV cache on an 80 GB card is a particular ratio. Nothing here is a claim about tensor-parallel serving, other engines, or smaller cards.

**Prefill-dominated by construction.** Prompt tokens outnumber generated tokens by more than 200:1 in every cell. That is the agentic shape I set out to measure, and it is also the shape that maximises what prefix reuse can do. A decode-heavy workload would show a much smaller gradient, and I have not measured one.

**n=3 per cell, one session.** Eighteen cells is enough to establish monotonicity and to show the bands are separated; it is not enough to characterise the tails. The campaign ran in a single ~1h20 session on one rented node for about $5.70. The calibration that fixed the H1 traffic parameters was cross-checked against a prior session and against a simulator matrix, but the published numbers come from one campaign.

**Sampling is not fully greedy.** Requests are sent with `temperature: 0.0`, but the model's own `generation_config.json` supplies a `repetition_penalty` of 1.05 that the server applies, since the request does not override it. This affects which tokens get generated, not how many — the cap governs volume — so it does not touch the gradient, which is a prefill effect. It does mean "greedy decoding" would be the wrong phrase for what ran.

**No exact active-window figure.** Reported already, repeated here because it is the load-bearing caveat: the window is fixed, the idle tail is measured but its energy is not separable, because only aggregate power statistics were persisted per cell rather than the per-sample series. That is an instrumentation gap in the harness, logged as backlog rather than papered over. Everything published is window-based, and window-based understates the gradient.

**Cell ordering is not recoverable from the JSON.** The per-cell manifests carry no timestamp field. Execution order can be reconstructed from the engine log, but the structured evidence alone does not carry it — a real gap in the evidence design, which matters if you want to rule out drift over the session.

**This is one axis of a calibration that is still open.** The phase-attribution design this builds on asks for isolation runs — a prefill-only workload and a decode-heavy one — to characterise how the two apportionments behave at the extremes. This campaign varies cache hit-rate instead, and gets six cells per band with clean separation. It constrains the question from one side. It does not close it.

## Reproducing the numbers

Everything above comes out of evidence committed in the repo. The headline table:

    python3 analysis/tokj_matrix.py \
      --matrix-dir gpu-evidence/session-20260711/matrix-20260711

That prints per-cell and aggregate tok/J, the phase energies, and the divergence for all eighteen cells. The active-time and phase-duration tables in this post are derived from the `phase_timeline.samples` array inside each cell's `inferscope.json` — one sample per second, carrying the engine's cumulative token and phase-time counters, so the point where they stop advancing is the end of work and the differences across the window are the phase durations.

The measurement tooling is [inferscope](https://github.com/MicheleCampi/inferscope): KV-cache hit-rate from the engine's Prometheus endpoint, energy from NVML's cumulative counter, and the two-basis phase apportionment whose divergence is Result 2. The [experiment repo](https://github.com/MicheleCampi/agentic-kv-energy-experiment) carries the protocol, the workload generator, the orchestrator, the calibration runs that fixed the H1 parameters, and the raw session evidence including the engine logs.

## What I take from it

The first result is the one that fits on a slide: on an agentic workload, going from cold prompts to 93% prefix reuse buys 69% more tokens per joule, and the gradient holds under failure traffic.

The second is the one that changed how I read my own instrumentation. I built the two-basis apportionment because I could not measure phase energy directly and did not want to pretend otherwise. What this campaign showed is that the choice of basis is not a wash — one of them is anchored to a quantity that cannot respond to the optimisation being applied. If I had shipped only the token-share number, I would have published a figure that stays at 0.996 while the thing it claims to describe moves by an order of magnitude, and nothing in the output would have flagged it.

That is the argument for publishing disagreement as a first-class signal rather than resolving it silently. The gap between two imperfect estimates told me something neither estimate could.
