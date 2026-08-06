---
layout: post
title: "Releasing the GPU while your agent waits is the obvious move. It saves nothing."
subtitle: "An agentic trajectory leaves the GPU allocated and idle for 2.3% to 37% of its span depending on how slow the tools are. The cost of generating does not move at all — but the obvious policy for reclaiming that idle time repays on none of fifteen measured cells."
date: 2026-08-06
categories: observability systems-engineering llm-inference
tags: rust llm profiling observability gpu a10 inferscope vllm agentic cost scheduling
---

An agentic loop spends a lot of its life not generating. It calls a tool, waits for the answer, thinks, calls another. On a rented GPU that waiting is not free: the card is allocated for the whole trajectory whether or not it is producing tokens, and the invoice is per hour of allocation.

There is an obvious platform response to that. If a tool call is going to take five seconds, release the GPU and reclaim it afterwards. Scale-to-zero, preemption, whatever the mechanism is called in your stack — the shape of the idea is always the same, and it always sounds right.

It is wrong across the entire range of tool latencies the literature reports, and the reason is a number nobody publishes: how long it takes to get the GPU *back*.

This post measures both sides. Fifteen cells on a single A10, sweeping tool latency across two orders of magnitude, with the cost of a trajectory decomposed into what was spent generating and what was spent waiting. The decomposition is the result I would put on a slide. The falsified policy is the one I would want a platform engineer to read.

## What is being varied, and what is being held still

Tool latency is a parameter *I* chose, which means the honest thing to publish is a curve rather than a point. Four values — 0.2, 0.5, 2.0 and 5.0 seconds per tool call — chosen to cover the interval the source characterisation reports for agentic workloads (tools occupy 2–29% of trajectory time, with the top of that range coming from GAIA-style tasks). Anything outside that interval would be measuring a workload nobody runs.

Three things are held still, and each one is why the curve is readable:

**The trajectory has fixed structure.** Four LLM calls, three tool calls, 192 generated tokens per call, declared seeds. This is a deterministic replay, not a live agent — and that is a measurement decision with a cost I paid before making it. Driving the sweep with a real Deep Agents loop was the first attempt: same prompt, same model, `temperature: 0.0`, three runs produced trajectory spans of 7.4s, 12.7s and 34.7s with four to eight steps. The model decides the shape, so both the numerator and the denominator of any ratio move for reasons unrelated to the parameter being swept. The replay's three repetitions at the same latency span 11.1 / 10.9 / 11.0 seconds: 1.8% spread against a factor of 4.7.

**The LLM span does not move.** Across all fifteen cells the generating portion holds at 25.5 seconds. Only the added tool time varies. If that had drifted, the curve would be measuring two things at once and would mean nothing.

**One card, one model, one engine.** vLLM 0.23.0, Qwen2.5-7B-Instruct, a single A10 24GB PCIe on Lambda at $1.29/h, measured 4 August 2026, `--enforce-eager` as the same matrix invariant used in [the energy campaign on this workload](/observability/systems-engineering/llm-inference/2026/07/30/agentic-kv-energy.html).

## Result 1: the cost of generating does not move

The unit is one trajectory: 768 generated tokens, identical in every cell. What changes is only how long the GPU sits allocated between those tokens.

Prices are derived at a declared $1.29/h — the rate actually paid, not a list price — and computed over the *trajectory span*, from the first step's start to the last step's end. Not over the sampling window. That distinction is load-bearing and I come back to it below.

| Tool latency | Non-generating fraction | Packing bound | $/M generated tokens |
|---|---|---|---|
| 0.2 s | 2.30% | 1.02 | $12.16 |
| 0.5 s | 5.55% | 1.06 | $12.62 |
| 2.0 s | 19.01% | 1.23 | $14.72 |
| 5.0 s | 37.01% | 1.59 | $18.91 |

Three replicas per row at declared seeds; the spread across replicas is 0.006 percentage points, which is the engine's own jitter and nothing else.

Now the decomposition, which is the part that makes the curve hard to argue with:

| Tool latency | Cost of generating | Cost of waiting on tools |
|---|---|---|
| 0.2 s | $0.009127 | $0.000215 |
| 0.5 s | $0.009155 | $0.000538 |
| 2.0 s | $0.009158 | $0.002150 |
| 5.0 s | $0.009147 | $0.005375 |

**The cost of generating is flat within a 0.5% band across all fifteen cells — $0.00911 to $0.00916 per trajectory.** Same tokens, same GPU, same model, same work. Meanwhile the cost of waiting grows by a factor of 25. The entire +56% in $/M token across the sweep is the GPU being allocated and not generating.

I quote the band rather than a rounded single figure on purpose. The claim is "this quantity does not move", and *how much* it does not move is the claim.

## Result 2: the obvious policy repays on none of fifteen cells

Call the policy P1: release the GPU on any tool segment longer than the price of getting it back. Saving is `Σ(d − C)` over segments with duration `d > C`, where `C` is the re-entry cost.

`C` is not a guess. An [earlier eBPF probe](https://github.com/MicheleCampi/vllm-coldstart-probe) measured vLLM cold start at roughly 18 seconds on this class of hardware, decomposed far enough to know that kernel I/O is only ~7% of it and the rest is GPU warm-up. There is a separate finding that the same start can take 27s or 96s depending on conditions I have localised but not root-caused, so 18s is the optimistic end.

The longest tool segment anywhere in the sweep is 5.0 seconds.

**P1 saves 0.000s on every one of the fifteen cells.** Not "a small amount". Zero, by construction: no segment in the interval the literature reports comes within a third of repaying an 18-second re-entry. Break-even would sit somewhere past 18 seconds per tool call, which is outside the published range of agentic tool latencies entirely.

The domain matters and travels with the claim: this holds for single-tenant serving billed at hourly granularity. If the freed GPU immediately serves something else, and if your billing granularity is fine enough to notice, the arithmetic changes. Neither is true of a rented A10.

So the idle time is not worth freeing. It is worth *filling* — which is what the packing bound in the first table quantifies.

## The bound, and why it ships with an error bar

`1 / (1 − f_nongen)` says how many trajectories one replica could host if the generating segments never contended for the card. At 2.0 s/tool that is 1.23. It is an upper bound under *declared non-interference*, and real batching does not work that way — which is why the number travels with that sentence attached rather than as a capacity figure.

But there is a second reason it should not ship as a bare number, and it cost three of the fifteen cells to find.

Real tools do not all take the same time. So one cell of the sweep was repeated at a coefficient of variation of 0.5 on the tool sleep — lognormal, parametrised by arithmetic mean and CV so that both flags mean what they say, and bit-identical to the deterministic cells at CV 0.

| Cells | Mean f_nongen | Spread across replicas |
|---|---|---|
| 2.0 s/tool, CV 0 | 19.01% | **0.006 pt** |
| 2.0 s/tool, CV 0.5 | 18.95% | **4.09 pt** |

The mean does not move — 0.06 points, nothing. The spread grows by three orders of magnitude. The curve is unaffected by tool-latency variance, exactly as the design predicted; what variance moves is how much you can trust any single observation of the bound.

So the bound at 2.0 s/tool is not 1.23. It is **1.23 ranging 1.20 to 1.26 under realistic tool variance**, and on price that reads as $15.02 / $14.81 / $14.30 per M token against ±0.03% for the deterministic cells.

Three cells is what it costs to publish a limit with its dispersion instead of a limit.

## A reading constraint I nearly published past

The profiler prints a `$/M gen tokens` figure computed over the *sampling window*. The window is an instrument parameter — I chose it, sized per cell from a calibration trajectory — and the three CV cells share the same 38-second window.

They print $17.7069, $17.7068 and $17.7068. Indistinguishable. Their `f_nongen` differs by 4.09 points.

The tool is not wrong: over a window you actually occupied, that is what the window cost. But it answers a different question from "what does this trajectory cost", and reporting it as though it were the second would have hidden the very dispersion the CV cells were spent to find. Every per-token figure in this post is recomputed over the trajectory span.

The generating/waiting split the tool prints is *not* affected — it is window-independent and reproduces the measured `f_nongen` exactly, 20.7% / 19.6% / 16.6% across those three cells.

## What this does not show

**It is not a price you can compare to an API listing.** $12–19 per million tokens comes from single-tenant profiling on one rented A10 with one trajectory in flight. No multi-tenant batching, no margin, no scale. It is a cost *attribution* over a run, not a quote.

**It is not real agentic traffic.** It is a deterministic replay of the traffic shape, anchored to published distributions. A live agent arm was run alongside as an anchor at the same latency, and its trajectories land in the region the replay describes — 31.5% against 36.3% at 2.0 s/tool, with the gap explained by the model choosing two tool calls where the replay pins three. Two arms, each proving what it can.

**One model, one card, one shape.** Qwen2.5-7B on an A10, four LLM calls and three tools per trajectory, 192 tokens per call. The generalisation on offer is the design, not the number.

**The packing bound has not been tested against a running engine.** It says what a replica could host under non-interference. Whether the idle windows actually get filled is a different experiment, [now designed with its falsification criterion stated in advance](https://github.com/MicheleCampi/vllm-coldstart-operator/blob/main/docs/adr/0010-a-measured-concurrency-bound.md) — the prediction being that N concurrent trajectories should show a time-averaged `num_requests_running` of `N × (1 − f_nongen)` rather than N. That runs in September.

**The re-entry price is one measurement with known variance.** ~18s on this hardware class, with a localised-but-not-root-caused case where the same start took 96s. P1 = 0 gets *more* robust as C grows, so the uncertainty is on the favourable side of the claim — but it is uncertainty.

## Reproducing the numbers

Every figure above regenerates from evidence committed in the repo: sixteen cell directories, eight files each, carrying the steps-file, its metadata, the profiler report, the exact argv that produced it, the derived cost and the decision-arm output. The argv are in the evidence because a cell that cannot be re-run from its own record is not evidence.

    python3 driver/analyze_cost_decision.py \
      validation-results-a10-cost/lat*/inferscope.json \
      --reentry-secs 18

That prints the non-generating fraction, the packing bound and the P1 saving for all fifteen cells. `--reentry-secs` has no default on purpose: the value has to be visible in the command that produced the numbers.

Design decisions were written down [before the node was switched on](https://github.com/MicheleCampi/agentic-kv-energy-experiment/blob/main/PROTOCOL.md) and the results sit beside them unedited, including the falsified one. Measurement by [inferscope](https://github.com/MicheleCampi/inferscope); the whole campaign cost $0.55 of GPU time.

## What I take from it

The decomposition is the part that transfers. On this workload the cost of generating is a constant — same tokens, same joules of useful work, whatever the tools are doing — and everything else on the invoice is allocation. That reframes the platform question: not "how do I make generation cheaper" but "how much of what I pay for is generation at all".

The falsified policy is the part I would not have got any other way. Releasing the GPU during tool calls is the intuition everyone has, including me when I designed the experiment, and the only reason I can say it does not pay is that the re-entry price was measured rather than assumed. A design that could only confirm would have produced a number and no information.
