---
layout: post
title: "Four GPUs, two sockets, one workload that didn't need any of it."
subtitle: "What inferscope showed me when I gave it more hardware to look at — and what it still couldn't resolve."
date: 2026-07-25
categories: observability systems-engineering llm-inference
tags: rust llm profiling observability gpu multi-gpu tensor-parallel inferscope a40
---

I rented four GPUs to validate multi-device profiling. By the end of the afternoon I had learned that tensor parallelism at 7B buys nothing I could measure, that two of my four GPUs were paying rent without doing work, and that my own profiler had been reporting the average of four things that needed to be reported separately.

This is a follow-up to [the H100 piece](https://michelecampi.github.io/observability/systems-engineering/llm-inference/2026/06/05/h100-profiler-hardware-utilisation.html) earlier in this series. The story this time is wider than one bug: four surprises, only one of which I planned for — plus one thing the session could not answer, which I'm keeping in the article because the shape of that gap is itself the point.

## Why I was running this at all

inferscope is a Rust profiler I've been building for LLM inference engines. It samples `/proc` for the host process and NVML for the GPU, correlates both with per-token timing, and emits one report. By the end of May it had been validated end-to-end on three single-device configurations: CPU baseline, RTX L4, and H100 SXM. The wrapper-PID bug discovered on L4 had been fixed in v0.2.1 and re-validated on H100.

What it had not been used for is multi-device. Production inference is rarely single-GPU. Models above ~30B at decent quantisation don't fit on one card; tensor-parallel serving is the default for anything in the 70B class. A profiler that only sees device 0 is solving half the problem.

The validation plan was straightforward: rent four GPUs, run llama.cpp with `--tensor-split` across all of them, and check that inferscope correctly enumerates and samples all four. Whatever extra I learned along the way, I told myself, would go in the article.

I learned more along the way than I planned to.

## The first surprise: the hardware isn't what the brochure said

I rented four NVIDIA A40s on RunPod. The dashboard said "4× A40, 192 GB VRAM, $1.76/hr". `nvidia-smi -L` confirmed four devices, 46068 MiB each, 300 W power limit, driver 565.57.01. What `nvidia-smi topo -m` showed was less reassuring:

```
        GPU0    GPU1    GPU2    GPU3    CPU Affinity    NUMA Affinity
GPU0     X      PXB     SYS     SYS     0-23,48-71      0
GPU1    PXB      X      SYS     SYS     0-23,48-71      0
GPU2    SYS     SYS      X      PXB     24-47,72-95     1
GPU3    SYS     SYS     PXB      X      24-47,72-95     1
```

The legend matters, and it's worth quoting the tool's own definitions rather than paraphrasing from memory. `PXB` is a connection traversing multiple PCIe bridges *without* traversing the PCIe host bridge. `SYS` is a connection traversing PCIe **as well as** the SMP interconnect between NUMA nodes — QPI/UPI, through the host. The NUMA affinity column confirms the split: GPU0 and GPU1 on node 0, GPU2 and GPU3 on node 1.

What I had rented, in physical reality, was not "four GPUs". It was "two pairs of two GPUs". Communication within a pair stays on the PCIe fabric; communication across pairs goes through the host.

This is not a RunPod quirk. It's the default shape of multi-GPU on rented infrastructure that isn't NVLink. NVLink-bonded sets exist — they're what makes 8×H100 SXM sensible — but they're a different price tier. Anyone renting 4 GPUs on Community Cloud at consumer-adjacent prices is renting this topology.

So the question became: how much does it matter? Hold that thought — the honest answer, at the end of this session, is that at this model size I couldn't resolve it.

## The setup that wasn't supposed to take this long

My first plan was to use vLLM. I `pip install vllm`'d, ran `vllm serve` with `--tensor-parallel-size 2`, and watched the engine die.

The error was clear once I found it: `The NVIDIA driver on your system is too old (found version 12070).` The vLLM wheel bundled a PyTorch build expecting a newer CUDA runtime than the rented A40 host driver supported. This was not fixable from inside the container: updating the host driver requires privileges I didn't have, and downgrading PyTorch inside vLLM's environment was a fight I didn't want to pick on an hourly clock.

I had a known-good alternative: llama.cpp b9165 with `--tensor-split`, the same engine I'd used on L4 and H100. Less narrative shine than "real production vLLM", but the question I came to answer was about inferscope and multi-device sampling.

A second snag: downloading Qwen 2.5 72B Q4 in parallel from HuggingFace hit unauthenticated rate limits and truncated all twelve shards. I fell back to Qwen 2.5 7B Q4 — the same model I'd used on H100. That fallback is the single most consequential thing that happened all afternoon, and I'll come back to it.

The working command:

```bash
/path/to/llama-server \
  --model /path/to/qwen2.5-7b-instruct-q4_k_m-00001-of-00002.gguf \
  --n-gpu-layers 999 \
  --tensor-split 1,1,1,1 \
  --ctx-size 4096 \
  --port 8080
```

inferscope v0.2.1 ran against this server with `--include-descendants --gpu`, sampling every visible device via NVML at a 50 ms tick.

## Surprise #2: at 7B, two more GPUs bought nothing I could measure

I ran two configurations of the same model, on the same hardware, with the same prompt and the same decode length. TP=2 used `CUDA_VISIBLE_DEVICES=0,1` — the same-socket pair. TP=4 used all four, which means crossing the socket boundary.

| | TP=2 (same-socket) | TP=4 (cross-socket) |
|---|---|---|
| Throughput | **106.33 tok/s** | **105.77 tok/s** |
| TTFT | 65.65 ms | 61.93 ms |
| Inter-token p50 | 9.31 ms | 9.39 ms |
| Inter-token p95 | 9.71 ms | 10.56 ms |

Adding two more GPUs changed throughput by −0.5%. p50 inter-token latency moved by under 1%. TTFT was slightly *faster* on TP=4, which is the wrong direction if you expect a serial parallelism penalty.

Here is the part where I have to be careful with my own data. **Each arm is a single request of 128 tokens, about 1.26 seconds of wall time.** With n=1 per arm and no repetitions, I have no variance estimate. A −0.5% throughput delta is not distinguishable from run-to-run noise — and neither is the TTFT delta in the other direction. The defensible claim is narrow: *on this hardware, at this model size, doubling the device count produced no throughput gain I could observe in a single sample.* It is not "TP=4 does not scale", and it is not a measurement of the cross-socket penalty either. Both of those would need repetitions I didn't buy.

What the result does do is match the mechanism you'd predict. Tensor parallelism pays off when per-device compute per layer is large compared to device-to-device synchronisation per layer. For Qwen 7B at Q4 — a 4.7 GB model with hidden dimension 3584 — spreading the same work across more devices means more synchronisation hops per token, eating into the time saved by doing less arithmetic per device.

There is a model size where 4-way tensor parallelism on this topology starts to win. It's not 7B. I couldn't measure where it is, because the larger model wouldn't download. I have one adjacent data point, from the H100 runs: Qwen 32B Q4 ran at 68.74 tok/s where Qwen 7B Q4 ran at 230.38 tok/s on the same card. That is a **measured throughput ratio of 3.35×**, not a ratio of work per token — the parameter-count ratio is about 4.6× — but it's the direction where per-device compute starts to swamp synchronisation overhead.

For 7B-class workloads, the practical read is that four GPUs is not obviously better than two, and probably worse than one card with adequate VRAM.

## Surprise #3: llama.cpp splits asymmetrically, and it matters

The aggregate numbers said TP=4 changed nothing. The per-device numbers said something more interesting:

| GPU | Socket | VRAM max | SM mean | Power mean |
|---|---|---|---|---|
| 0 | 0 (with 1) | 1.61 GiB | **19.2%** | 138.9 W |
| 1 | 0 (with 0) | 1.41 GiB | 16.4% | 130.9 W |
| 2 | 1 (with 3) | 1.41 GiB | 16.6% | 129.8 W |
| 3 | 1 (with 2) | **1.92 GiB** | **22.2%** | **139.4 W** |

The first and last devices in topology order carry more than the two in the middle. Measured against the mean of the middle pair:

- GPU0: **+14% VRAM, +16% SM, +6.6% power**
- GPU3: **+36% VRAM, +35% SM, +6.9% power**

Two things are worth separating there. The direction is consistent across all three signals — boundary devices work harder, on every axis. The *magnitude* is not: memory and occupancy move by 14–36%, while power moves by under 7% on both. That's a reminder that power draw at this utilisation is dominated by the idle floor rather than by the marginal work, which is exactly the effect the next section is about.

The reason for the asymmetry is in how llama.cpp partitions the model. `--tensor-split 1,1,1,1` distributes transformer layers evenly. It does **not** distribute the embedding layer or the output projection (lm_head). The embedding lives on GPU0, where prompt tokens enter the pipeline; lm_head lives on the last device, where logits are sampled before streaming back. Both are among the largest single tensors in the model — I did not measure their on-device footprint separately, so I'm not going to attach a number to them, but they are the structural explanation for a boundary-heavy distribution.

The practical implication: when capacity-planning for tensor-parallel serving on llama.cpp, don't provision on the assumption of an even split. On a tightly-packed deployment serving long contexts, the first and last devices are the ones that run out of headroom first. The middle devices are not your constraint.

## Surprise #4: idle GPUs cost real money

In the TP=2 run, two of the four GPUs were unused. `CUDA_VISIBLE_DEVICES=0,1` restricted llama.cpp to the same-socket pair; GPU2 and GPU3 saw zero compute for the whole run.

What they did not see was zero power.

| GPU | State | SM mean | Power mean |
|---|---|---|---|
| 0 | active | 33.1% | 148.0 W |
| 1 | active | 37.5% | 152.5 W |
| 2 | idle | 0.0% | **34.1 W** |
| 3 | idle | 0.0% | **32.3 W** |

66.4 watts combined, for two devices that did literally nothing. That's the A40 idle floor — the cost of the chip being powered on with memory clocks alive. Against the 300 W power limit these cards report, idle is about 11% of maximum.

The dollar figure is worth stating carefully, because it is **derived from list price, not measured**: at the $1.76/hr I paid for four cards in May 2026, two idle cards are ~$0.88/hr buying zero throughput, which annualises to roughly $640 a month of paid idle if you left it running continuously. I did not run it continuously; that's arithmetic on a rate card, and rate cards change.

The principle generalises even where the number doesn't: every NVIDIA datacenter GPU has an idle power floor, and the only way out is to not have the device in your container at all. Renting "a few extra in case" pays full rent for partial work.

## What this told me about my profiler

So far this article has been about the workload. The fourth surprise is about inferscope itself.

When I read the top-level GPU section of the JSON for the TP=2 run, I saw this:

```
sample_count: 108
device_count: 4
memory_used_max_bytes: 3076915200    // 2.87 GiB
utilization_mean_percent: 17
power_mean_milliwatts: 91704         // 91.7 W
```

These numbers are not wrong. There were 27 ticks × 4 devices = 108 samples, and `utilization_mean_percent: 17` is genuinely their arithmetic mean. The math is right.

The numbers are also not useful. "Mean SM 17%" is averaging two GPUs at 33% and 38% with two GPUs at 0%. The system isn't running at 17%; half of it is working and half of it is sitting cold. "Max VRAM 2.87 GiB" is device 1 — the most-loaded card in this specific configuration. It is not the total VRAM in use across the four devices, and it is not what a reader would naturally assume it was.

This is a representation problem, not a sampling problem. v0.2.1's NVML backend correctly enumerated all four devices, correctly produced one sample per device per tick, and correctly tagged each sample with `device_index`. The data was there. The aggregation discarded it.

That fix shipped in v0.3.0, recorded in ADR-007: the JSON report keeps the cluster-level aggregate — still useful for "is the whole system busy?" — and adds a `per_device` block alongside it, for "which devices, doing what, drawing how much power?". The text report switches to per-device tables when `device_count > 1`; single-device runs render as they did in v0.2.x. Every per-device number in this article was recovered by post-processing `gpu_timeline.samples[]` by hand, which is precisely the work the new schema removes.

The pattern from the H100 piece holds: a profiler that aggregates across a dimension you care about hides the answer you wanted.

## A different engine, same profiler

Throughout this article the engine in front of the profiler has been llama.cpp. That choice was forced by the driver mismatch above. Fine for one session; not fine as a standing claim that inferscope works with vLLM. So I rented a fresh H100 SXM where the driver was current, installed vLLM 0.21 against Qwen 2.5 7B in AWQ, and pointed inferscope at it without changing a line of code. It worked the first time.

Comparing single-GPU numbers on the same hardware, llama.cpp Q4_K_M against vLLM AWQ — and this time comparing a *single* run on each side, not a best-of assembled per row:

| | llama.cpp (Q4_K_M) | vLLM 0.21 (AWQ) |
|---|---|---|
| TTFT (warm) | 39.71 ms | **20.71 ms** |
| Throughput | 230.38 tok/s | **237.99 tok/s** |
| SM mean | 46% | 57% |
| VRAM max | 5.17 GiB | **73.53 GiB** |

vLLM roughly halves TTFT (−47.8%), edges out throughput by 3.3%, and uses **14× the VRAM** — that last number is the aggressive KV cache pool vLLM allocates to support continuous batching. For single-request serving llama.cpp is competitive. For batched concurrent serving, which is what production looks like, vLLM is built for a workload llama.cpp wasn't.

There is a row I deliberately left out of that table: power. I had originally written up a clean "vLLM draws ~19% less power per token" claim, and it doesn't survive contact with the data. Computing energy per token from the aggregate power mean gives −8.0% for vLLM; computing it from only the samples where the GPU was actually busy gives −15.5%. Two defensible estimators, on the same two runs, disagreeing by nearly a factor of two — over 13 GPU samples per run, on requests lasting well under a second. That spread isn't a finding about vLLM; it's my sampling window being too short to resolve the question. The honest version is that I don't have a per-token energy comparison from this session, and reporting either number as *the* answer would be choosing whichever one told the better story.

## The startup pattern that only shows up if you run more than once

One more thing, and it's the one I'd have missed with a single run.

Across three vLLM runs the TTFT was 22.84 ms, then 651.78 ms, then 20.71 ms. The obvious story — "the first request to a cold server is slow" — is wrong, and so was the second story I told myself. The server logs settle it, and they were sitting in my own evidence archive the whole time.

Here is what they show. In both sessions, CUDA graph capture finishes during engine init, before the server accepts any traffic: `Graph capturing finished in 3 secs, took 0.63 GiB`. And `torch.compile` runs on *both* starts, at essentially the same cost — 15.93 s on the first-ever start, 16.62 s after the restart. There is no compile-cache shortcut on restart, so a graph-capture explanation has nothing left to stand on.

What both logs do record, on the first inference after each engine start, is this:

```
WARNING [jit_monitor.py:103] Triton kernel JIT compilation during inference:
_compute_slot_mapping_kernel. This causes a latency spike; consider extending warmup
```

vLLM diagnoses itself. The first request after an engine start pays Triton JIT compilation of a kernel that warmup didn't cover, and vLLM's own monitor flags it as a latency spike.

Which leaves the awkward part. My run labelled `cold` reports 22.84 ms — because the log shows *another* request had already hit that server minutes earlier and absorbed the JIT cost. It was never a cold measurement. The 651.78 ms run is the only one of the three where I actually profiled a first inference, and I had it filed under the wrong name for two months.

So the operational point survives, sharpened: **the first request after any engine start pays a compilation cost, and a benchmark that sends one throwaway request first will never see it.** That is not a hypothetical failure mode. It is what happened to my own measurement, in my own archive, until I read the server log instead of the report.

The raw reports and both server logs are in the [inferscope repo](https://github.com/MicheleCampi/inferscope/tree/main/validation-results/vllm-h100-2026-05-24), including a correction note on the summary file that got this wrong first time.

## Closing

I came to this session expecting to validate that inferscope sampled four GPUs correctly. It does. I came expecting to learn whether four-way tensor parallelism scaled at 7B. It didn't, in the one sample I took — and one sample is what I'm entitled to claim. I did not come expecting to learn that my report format needed rethinking, or that a short run makes energy numbers unresolvable, or that one of my own runs had been mislabelled cold for two months.

The arc that started with a `/proc`-only profiler missing a GPU bug ends with the same profiler, against a different engine on different hardware, surfacing a startup pattern that single-shot benchmarks don't see. The data the profiler captures is fine. The question is which slice the report makes visible — and whether anyone runs the engine long enough for the interesting slice to exist at all.

If you've run tensor-parallel inference on rented multi-GPU and measured different scaling numbers — particularly if you have a model size where 4-way TP genuinely pays off on a non-NVLink topology — I'd be interested to compare notes. The [repo](https://github.com/MicheleCampi/inferscope) is public, ADR-007 included.
