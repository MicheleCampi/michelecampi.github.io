---
layout: post
title: "The client measured the cost. Only the per-device view measured the trade-off."
subtitle: "Profiling aggregated vs disaggregated LLM serving on 2×H100 — what each view could and couldn't see."
date: 2026-07-19
categories: observability systems-engineering llm-inference
tags: rust llm profiling observability gpu h100 inferscope vllm disaggregation
---

Disaggregated serving is the direction the industry is moving: split the compute-bound prefill phase from the memory-bound decode phase onto separate GPUs, so each can scale on its own. NVIDIA Dynamo, vLLM's disaggregation support, Mooncake — the premise is sound at cluster scale. But most of the writing about it *is* about clusters: dozens of GPUs, heterogeneous pools, cross-node KV transfer over InfiniBand.

I wanted to know something narrower and more concrete. On the smallest setup where disaggregation is even possible — one node, two GPUs — what does it actually cost, and what does it actually buy? And can I see the answer from the outside, at the client, or do I have to look inside, at each device?

So I ran an A/B experiment and profiled both arms three ways.

## Setup

Everything is held equal except the one thing under test: how vLLM uses the two GPUs.

- **Hardware:** a single Lambda Cloud node, 2× H100 80GB SXM5. Identical for both arms.
- **Engine / model:** vLLM 0.21.0, Qwen2.5-7B-Instruct. Both arms run with `--enforce-eager`, so the comparison isn't muddied by CUDA-graph capture differences.
- **Arm A — Aggregated (TP=2):** one vLLM instance, tensor-parallel across both GPUs. The conventional setup.
- **Arm B — Disaggregated (1P1D):** a prefiller pinned to GPU0, a decoder pinned to GPU1, and a proxy in front. The KV cache the prefiller produces is handed to the decoder over LMCache/NIXL (`LMCacheConnectorV1`, producer/consumer roles). This is vLLM's own disaggregation example, used as-is.
- **Load:** `vllm bench serve`, random dataset, 200 prompts at a request rate of 3.6 req/s — identical for both arms. Three regimes, chosen to sweep the prefill/decode balance:
  - **prefill-heavy** — 7500 input / 200 output
  - **balanced** — 2000 / 2000
  - **decode-heavy** — 500 / 4000

## Method: three views

A single number lies by omission, so I measured three:

1. **Client-side**, from `vllm bench serve` — throughput, TTFT, TPOT, percentiles. What a user of the endpoint feels.
2. **Per-device GPU**, with [inferscope](https://github.com/MicheleCampi/inferscope) in `--sample-only --gpu` mode at 50 ms — utilisation, power, memory, *per device*, for the lifetime of the run. What the hardware is doing.
3. **The gap between them** — which is where the story turned out to live.

One honest caveat about the per-device numbers, because it changes how you should read them. inferscope sampled over a window deliberately longer than each benchmark (sampling ran 90–300 s; the benchmarks themselves ran 58–109 s). So *absolute mean power is diluted by idle-tail time and is not comparable across runs of different lengths.* I therefore don't lean on cross-run absolute means. What is clean and comparable — and what I report — is the **per-device ratio within a single run** and the **peak power**.

## Client results

| Regime | Arm | Out tok/s | Total tok/s | TTFT mean (ms) | TTFT p99 (ms) | TPOT mean (ms) | Failed |
|---|---|---|---|---|---|---|---|
| prefill-heavy | Aggregated | 693.0 | 26680 | 148.3 | 159.5 | 14.2 | 0 |
| | Disaggregated | 692.7 | 26667 | **442.7** | **1056.3** | 21.9 | 0 |
| balanced | Aggregated | 5242.7 | 10485 | 62.0 | 70.1 | 10.8 | 0 |
| | Disaggregated | 5160.7 | 10321 | 142.5 | 163.2 | 13.2 | 0 |
| decode-heavy | Aggregated | 8197.2 | 9222 | 43.4 | 55.1 | 10.7 | 0 |
| | Disaggregated | **7308.8** | 8222 | 79.6 | 109.4 | 13.8 | 0 |

Zero failed requests across all six runs — both arms are stable at this load. But disaggregation imposes a latency tax in every regime and never improves throughput:

- **prefill-heavy:** throughput is a dead heat (~26.7k total tok/s either way), but mean TTFT nearly triples (148 → 443 ms) and p99 TTFT blows out from 160 ms to **1056 ms** — the KV-transfer cost surfacing in the tail. TPOT +55%.
- **balanced:** throughput essentially tied (−1.6%), TTFT +80 ms, TPOT +22%.
- **decode-heavy:** the regime decode GPUs exist for, where disaggregation should look its best — and instead output throughput *drops 11%* (8197 → 7309 tok/s) and the run takes 12% longer, with TTFT and TPOT both still worse.

From the client's chair, then: at this scale disaggregation costs, in every regime, on every metric I measured. If I'd stopped here, the conclusion would be "don't bother." That conclusion is also incomplete.

## Per-device results

GPU0 is the prefiller, GPU1 the decoder (in the aggregated arm both run tensor-parallel).

| Regime | Arm | GPU0 — util / mean W / peak W | GPU1 — util / mean W / peak W | Power ratio |
|---|---|---|---|---|
| prefill-heavy | Aggregated | 14% / 182 / 509 | 17% / 182 / 494 | 1.00× |
| | Disaggregated | 36% / 368 / 624 | 62% / 448 / 705 | 1.22× |
| balanced | Aggregated | 66% / 360 / 481 | 75% / 358 / 473 | 1.00× |
| | Disaggregated | 5% / 167 / 250 | 63% / 414 / 654 | 2.48× |
| decode-heavy | Aggregated | 56% / 319 / 576 | 55% / 321 / 573 | 1.00× |
| | Disaggregated | 1% / 127 / 161 | 70% / 466 / 695 | **3.66×** |

The aggregated arm is perfectly symmetric: in every regime the two GPUs draw within a fraction of a percent of each other — 1.00× at the mean and near the peak alike. That is the signature of tensor parallelism: both GPUs do the same work in lockstep.

The disaggregated arm is asymmetric, and the asymmetry grows **monotonically with the decode share** of the workload: 1.22× → 2.48× → 3.66×. At the decode-heavy extreme the picture is stark — the prefiller sits at **1% utilisation and 127 W, essentially idle**, while the decoder runs at 70% and 466 W mean, peaking at **695 W**, within a few watts of the card's 700 W limit. One H100 pinned to the wall; the other parked.

## The finding

Put the two views side by side and the trade-off becomes legible in a way neither shows alone.

**The client sees only the cost** — a latency tax, no throughput win. **The per-device view sees what disaggregation actually does:** it isolates the phases. And at this scale, in the very regime where that isolation is supposed to pay off, what isolation produces is a *stranded prefiller* — a whole H100 at 127 W while its partner redlines.

What disaggregation buys here isn't speed. It's a clean, measurable separation of where the work goes — which is precisely the precondition for the thing that makes disaggregation worth it at scale: independently scaling the two pools (many prefillers feeding fewer decoders, or the reverse) so no GPU sits stranded. With one of each, you pay for the separation and can't yet collect on it.

That's the sentence I keep: the cost is visible from the client; the benefit — and, at 1P1D, the *un-collected* benefit — is visible only per-device. A throughput chart would have told me disaggregation lost. The power-asymmetry curve told me why I'd still reach for it the moment I had more than two GPUs to arrange.

## Limits and scope

I want to be precise about what this does and doesn't show, because it would be easy to over-read.

- **This is one node, two GPUs, one 8B model** — the smallest disaggregation topology that exists. Nothing here argues against disaggregation at the scale it's designed for: xPyD across many nodes, heterogeneous hardware, KV over a fast interconnect. The monotonic asymmetry curve is, if anything, the argument *for* that scale — the 3.66× stranding at 1P1D is exactly what independent pool-scaling exists to absorb.
- **KV transfer here is LMCache/NIXL on a single node.** Cross-node transfer characteristics will differ.
- **Request rate is fixed at 3.6 and the dataset is random.** A production trace with prefix reuse would shift the prefill economics.
- **`--enforce-eager`** leaves CUDA-graph performance on the table for both arms equally — it keeps the comparison clean but isn't a production config.

The claim is narrow and, I think, defensible: *at 1P1D on equal hardware, disaggregation is a latency-and-throughput cost, and the only way to see what you're paying for is to measure each device separately.*

## Closing

The result I keep coming back to isn't the 11% or the 3.66×. It's that the single most important number in the whole experiment — the stranded prefiller — appears nowhere in the client output. It exists only per-device.

That's the reason I build the tooling I build: the interesting part of an inference system is usually the part the endpoint can't show you.

Reproduction scripts for both arms, all six runs, and the raw inferscope JSON are in the [repository](https://github.com/MicheleCampi/vllm-disagg-profiling). Profiling by [inferscope](https://github.com/MicheleCampi/inferscope).
