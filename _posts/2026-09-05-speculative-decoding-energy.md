---
layout: post
title: "Speculative decoding pays before it accepts anything."
subtitle: "Eleven runs on an H100 looking for the acceptance rate where speculation stops being worth its energy. There isn't one — and the reason is not that the drafts were good."
date: 2026-09-05
categories: observability systems-engineering llm-inference
tags: rust llm profiling observability gpu h100 inferscope vllm speculative-decoding energy
---
Speculative decoding is sold on a simple trade. A small draft model proposes k tokens, the big one verifies them in a single forward pass, and whatever it accepts you got for the price of one step instead of k. The drafts it rejects are waste — a forward pass on the draft model plus a verification slot, producing nothing.

That framing suggests a break-even. Below some acceptance rate you are paying for more waste than you are saving, and speculation should cost you. Above it, it pays. Every tuning guide implies that line exists; the metrics vLLM reports — acceptance rate, mean acceptance length, tokens per second — are all measured as if finding it were the point.

None of those metrics is in joules, and the waste is a hardware cost. So I went to measure where the line is.

There isn't one. Not in the range I swept, and not for the reason I expected.

## Making acceptance a knob instead of a property

The hard part of this measurement is not the energy — [inferscope](https://github.com/MicheleCampi/inferscope) already reads NVML's hardware energy counter on the same clock as the engine's own counters. The hard part is that acceptance rate is normally a *property* of a (draft model, target model, workload) triple, not something you set. Two points on a sweep would differ in the draft model as well as in acceptance, and no energy difference could be attributed to either.

vLLM has a facility for exactly this that I have seen almost nobody use. `rejection_sample_method: "synthetic"` accepts draft tokens against a declared per-position rate vector instead of against the target's distribution:

```json
{
  "method": "draft_model",
  "model": "Qwen/Qwen2.5-0.5B-Instruct",
  "num_speculative_tokens": 5,
  "rejection_sample_method": "synthetic",
  "synthetic_acceptance_rates": [1.0, 1.0, 0.0, 0.0, 0.0]
}
```

It substitutes only the accept/reject *decision*. The draft proposal runs, the target verification forward pass runs, both unchanged — so the energy you measure is the energy of real speculative work, with acceptance turned into an independent variable. That is verified in the kernel rather than assumed: the `SYNTHETIC_MODE` branch in `rejection_greedy_sample_kernel` replaces the comparison against `target_argmax_id` and nothing upstream of it.

Two arms at each matched mean acceptance length. One is the minimum-variance schedule vLLM resolves internally from a scalar target — `[1.0, 1.0, 0.4, 0.0, 0.0]` for a mean of 3.4, the first positions always accepted and the rest never. The other is an explicit geometric vector at the same mean, which is closer to how real acceptance decays. If the crossover moved between them, dispersion would matter and the scalar knob alone would be an insufficient instrument.

## The setup

- **Hardware:** 1× H100 PCIe 80GB (Lambda), driver 580.105.08, vLLM 0.28.0.
- **Models:** Qwen2.5-3B-Instruct as target, Qwen2.5-0.5B-Instruct as draft, k=5. Not the pair I planned: Llama is gated, and within Qwen2.5 the small models carry `vocab_size=151936` while 7B and above carry 152064, so a 7B target has no compatible small draft — vLLM raises on the mismatch under `method="draft_model"`.
- **Load:** `vllm bench serve`, random dataset, 512 in / 256 out, `--ignore-eos`, 3 req/s, fixed seed, 1600 prompts so the load outlives the measurement window rather than finishing inside it.
- **Per run:** 60s warm-up unsampled, then a 180s energy window, then 45s cooldown and a server restart.
- **Eleven runs:** a no-speculation baseline, nine acceptance points, and the baseline again. Run order randomised under a recorded seed, so thermal drift could not correlate with acceptance length.

The discard criteria were written into the analysis script before the node was booted, and the script refuses to derive a curve if any of them fires. That is deliberate: a criterion applied by hand, with the GPU billing and ten runs already behind you, is a criterion that gets negotiated.

The session passed all of them. The two baselines came in 0.13% apart on energy per token — 43,030,674 mJ opening against 42,975,418 mJ closing — and realized acceptance length matched the configured value on all nine points, including the geometric arm at 1.9947, 2.9918, 3.9958 and 5.0025.

## No crossover

Baseline, no speculation: **328.744 mJ per committed token.**

| run | realized L | mJ/token | vs baseline |
|---|---|---|---|
| L=1.00 min-var | 1.00 | 294.855 | 0.897× |
| L=2.00 geometric | 1.99 | 263.631 | 0.802× |
| L=2.00 min-var | 2.00 | 267.035 | 0.812× |
| L=3.00 geometric | 2.99 | 252.725 | 0.769× |
| L=3.00 min-var | 3.00 | 249.929 | 0.760× |
| L=4.00 geometric | 4.00 | 243.737 | 0.741× |
| L=4.00 min-var | 4.00 | 243.225 | 0.740× |
| L=5.00 geometric | 5.00 | 235.296 | 0.716× |
| L=5.00 min-var | 5.00 | 237.746 | 0.723× |

Speculation is cheaper per committed token than not speculating at every acceptance length, from 10.3% cheaper at the bottom to 28.4% at the top. The line I went looking for is not in this range.

The falsification criterion, fixed before the runs, said that outcome would be the result rather than a failed campaign. It also means the instrument question is unanswered: comparing the two arms needed a crossover to compare, and there is none. The two arms track each other to within a percentage point at every point, which is suggestive and is not an answer.

## The point that is not a null result

Look at the first row again.

At L=1.00 the acceptance vector is `[0, 0, 0, 0, 0]`. Nothing is ever accepted. The counters confirm it: **130,576 speculation rounds, 652,880 draft tokens proposed, zero accepted.** Every draft in that run was computed by the 0.5B, shipped to the target for verification, and thrown away.

That run committed its tokens at **0.897× the baseline's energy**.

Speculating and accepting nothing is 10% cheaper per committed token than not speculating at all. The waste was real — six hundred and fifty thousand tokens of it — and the run was still ahead.

## Why

The mechanism is not in the speculation succeeding. It is in the shape of the forward pass.

vLLM computes a property called `uniform_decode_query_len`, and its docstring says what it is: *"A decode step submits one query for the newly sampled token plus one for each draft token."* Without a speculative config it is 1. With k=5 it is 6.

Decode is memory-bandwidth-bound. The dominant cost of a decode step is streaming the model weights from HBM into the compute units, and that cost is paid once per forward pass regardless of how many queries ride along in it. A verification step that carries six queries pays the same weight-streaming bill as a step that carries one, and amortises it over six — even when five of the six are discarded.

The draft model is not free: it is a 0.5B forward pass per round, and it is real energy. It is just smaller than the amortisation it buys on a 3B target.

Acceptance rate cannot see any of this. Neither can tokens per second. Both are throughput figures, and at zero acceptance both correctly report that speculation achieved nothing — while the energy counter says it saved 10%.

## Killing the alternatives

A 10% saving at zero acceptance is the kind of number that is usually an artefact, so I checked the ways it could be one before believing it. All of these are readable in the source or in the reports:

**The denominator does not include drafts.** Energy per token divides by `vllm:generation_tokens_total`, which counts committed tokens. Across all eleven runs the generated token count is constant within 1.32% (130,003 to 131,723), so the load did the same work in every arm and the counter is not inflating with acceptance.

**The scheduler budget is the same in both arms.** `_set_max_num_scheduled_tokens` sets it equal to `max_num_batched_tokens` and never subtracts the drafting delta; the delta it computes is used for validation only. And `draft_model` requires 1 additional slot per request, not k — the table in `SpeculativeConfig.max_num_new_slots_for_drafting` is explicit about it — against a budget of 8192.

**The load was identical in fact, not just nominally.** Both arms completed 648 of 1600 requests in 3 minutes 46 seconds. The arrival rate was the bottleneck, not the server, so mean concurrency and batch shape were the same on both sides.

**The KV cache differs, and not enough to matter.** The draft model's weights are in VRAM when the KV cache is sized — vLLM loads the model, then profiles peak memory, then allocates what is left — so the baseline has roughly 1 GB more KV cache than the speculative arms. Against the 65.5 GiB actually allocated, and a maximum concurrency of 465× where the real load ran in the tens, that difference never binds.

**The counters are internally consistent.** `draft_tokens / drafts` is exactly 5.0 in every speculative run, which is what a draft model should produce and what ngram would not.

## What this does and does not license

If you are deciding whether to turn speculative decoding on for a memory-bandwidth-bound target, the acceptance rate your draft model achieves is not the gate. On this hardware and this model pair it scales the benefit — 10% at nothing accepted, 28% at five — but it does not determine the sign. A draft model that turns out worse than you hoped does not put you behind where you started.

That is a different decision procedure than the usual one, which treats acceptance rate as a threshold to clear.

What it does not license is generalisation, and I want to be first to say where it stops:

- **One target/draft pair.** A 3B target with a 0.5B draft. The draft is a sixth of the target; on a 70B target with the same draft the amortisation is larger and the draft cost proportionally smaller, and on a target where the draft is a third of the size it could invert. Untested.
- **One device.** H100 PCIe, roughly 2.0 TB/s of HBM bandwidth. The whole mechanism is a bandwidth-versus-compute ratio, so an SXM part at 3.35 TB/s moves the balance by construction. Untested.
- **One workload.** 512 in, 256 out, 3 req/s, never saturated. A saturated server has different batch shapes and the amortisation argument changes.
- **Synthetic acceptance is not model acceptance.** This measures how energy responds to acceptance with everything else held fixed. It says nothing about what acceptance any real draft model achieves on your data.
- **One comparison I could not make.** The speculative servers' logs were overwritten per run and the instance released before I copied them, so I never compared the CUDA graph sizes captured in the two arms. The query-length difference *is* the speculation rather than a configuration artefact, but the comparison was not made and I am not going to pretend otherwise.

## Reproducibility

The protocol, the nine generated configs, the campaign script, the analysis script and its tests, and all eleven reports are in [`validation-results/adr-016-h100-spec/`](https://github.com/MicheleCampi/inferscope/tree/main/validation-results/adr-016-h100-spec). Every number in this post comes from a command run over those reports, not from reading a table.

The session cost about 90 minutes of an H100 and roughly five dollars. The instrument is [ADR-016](https://github.com/MicheleCampi/inferscope/blob/main/docs/adr/016-speculative-decoding-energy.md), which was written before any of it ran and is unedited apart from a dated validation entry.
