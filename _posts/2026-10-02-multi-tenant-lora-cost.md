---
layout: post
title: "Eight LoRA adapters cost 6–7% more per token. Skewing the traffic among them doesn't."
subtitle: "56 cells on an A10, asking whether the per-adapter counts that vLLM computes and never exports would tell a router anything about cost. They would not."
date: 2026-10-02
categories: observability systems-engineering llm-inference
tags: llm-inference vllm lora multi-tenancy llm-d gpu a10 inferscope energy
---
A vLLM server with LoRA enabled counts how many requests each adapter has running. It keeps the count in a dictionary keyed by adapter, and then exports only the keys.

At `v0.30.0`, `vllm/v1/metrics/stats.py:643` writes `running_lora_adapters[lora_name] = len(stats.running)`. The exporter turns that dictionary into `",".join(scheduler_stats.running_lora_adapters.keys())` and sets it as a label of `vllm:lora_requests_info`, a gauge whose value is the current time (`vllm/v1/metrics/loggers.py:1085-1096`). The counts are gone. The Rust frontend does the same: its label type is `LoraAdapterNames(pub BTreeSet<String>)`, a set of names joined by commas (`rust/src/metrics/src/scheduler.rs:123-128` on `main` at `0cbac6cd1`, 2026-10-02), and the Python side at that commit still keeps only the keys (`loggers.py:1155-1157`).

Downstream, the loss is inherited rather than repaired. llm-d's router splits the label into a `map[string]int` and writes zero for every name (`extractor.go:257`, `312` on `main` at `aa5526c`). Its LoRA scorer then discards even the zero, `_, active := m.ActiveModels[request.TargetModel]` (`lora_affinity.go:89`), and scores an endpoint 1.0 if the adapter is active there, 0.8 if there is room for one more, 0.6 if it is only waiting, 0.0 otherwise (lines 104-112). llm-d's inference simulator, which its README describes as built "to mimic the behavior of vLLM" so that schedulers can be tested without GPUs, joins `maps.Keys(snap.Running)` the same way (`pkg/engine/vllm/metrics.go:815` at `bf3f6a6`).

So a router cannot tell apart two pods serving the same adapters when one gives an adapter 90% of its traffic and the other 5%. The obvious move is to propose exporting the counts — a change across three projects. Before proposing it, I wanted to know whether the number would tell the router anything about cost.

## The question, fixed before the node

The protocol was written on 2026-09-20 and committed before any measurement existed. It states two hypotheses and the result that would falsify each:

- **H1 — the number of adapters costs.** Cost per generated token at eight adapters exceeds cost at one. Falsified if the difference stays under 5%.
- **H2 — the imbalance costs, at fixed N.** At two, four and eight adapters, sending most of the traffic to one adapter costs differently per token than an even split. Falsified if the difference is smaller than the band of repetitions.

H1 is the control, without which H2 cannot be read. H2 is the question. If imbalance costs, the router is choosing between endpoints on a dimension it cannot see, and the scorer's own comment already marks where a fix would go: "This may change later if vLLM adds native support" (`lora_affinity.go:92-94`). If it does not, the three projects were right to drop the signal.

Two dry runs on the node fixed what could not be fixed away from it, and what they settled was added to the protocol as dated amendments, leaving the original text as it was. Which cost per token decides was among them, and it matters more than it looks; it comes up again below. The last commit to the protocol is dated 2026-09-30 17:33 UTC; the first cell's measurement started at 19:56 UTC.

## The setup

One NVIDIA A10, vLLM 0.30.0 from PyPI, `Qwen/Qwen2.5-7B-Instruct`, and eight public adapters from eight different authors, all rank 16 over the same seven projections, pinned by revision.

What could vary cost for a reason other than multi-tenancy is held fixed, each choice justified from the source in the protocol. `--max-loras 8` in every cell, because vLLM preallocates the LoRA weights per slot, `torch.zeros(max_loras, ...)` (`vllm/lora/layers/base_linear.py:134-135`, `145-146` at `v0.30.0`): N=1 carries seven empty slots, and the memory footprint is the same in every cell. `--enforce-eager`, because vLLM has an option to capture separate CUDA graphs "for different counts of active LoRAs (powers of 2 up to max_loras)" (`vllm/config/lora.py:68-73`), which would tie cost to a capture detail; with `--enforce-eager` vLLM will "disable CUDA graph and always execute the model in eager mode" (`vllm/config/model.py:242-244`).

The load is vLLM's own benchmark, `vllm bench serve`, so latency and throughput are computed by vLLM's code rather than by something similar written for this. Every request is 256 tokens in and 256 out, with `ignore_eos`, so every cell does the same generative work whichever adapter serves it. The imbalance needs no patch: the benchmark copies the list of adapters as given, `lora_modules_list = list(lora_modules)`, and assigns it round-robin, `lora_modules_list[i % len(lora_modules_list)]` (`vllm/benchmarks/serve.py:917`, `922`), so an adapter listed 21 times in a 28-entry list gets 75% of the requests, interleaved through the cell rather than in a burst.

Seven configurations — uniform at N = 1, 2, 4, 8, and 75% to one adapter at N = 2, 4, 8 — at concurrency 128, the lowest level that reached 90% of the highest throughput in the dry run, and again at 64. Four repetitions each, in four rounds with the order rotated: 56 cells in one node session, every one kept, none rerun. Energy is the delta of the GPU's NVML energy counter over each cell's window, read by [inferscope](https://github.com/MicheleCampi/inferscope) (`crates/is-sysmon/src/gpu_nvidia.rs:179`, `201-202` at `acd21ec`).

## The count costs

Energy per generated token, net of idle:

| Concurrency | N=1, J/token | N=8, J/token | Margin | Bands N=1, N=8 |
|---|---|---|---|---|
| 128 | 0.107749 | 0.115340 | +7.05% | 1.45%, 0.67% |
| 64 | 0.155812 | 0.165093 | +5.96% | 0.50%, 0.62% |

Both above the 5% threshold, with no band wider than 1.45%. H1 is not falsified. Output throughput at N=8 is 5.80% lower than at N=1 at concurrency 128 and 5.70% lower at 64.

The cost does not grow evenly with the count. Against N=1 at the same concurrency:

| Concurrency | N=2 | N=4 | N=8 |
|---|---|---|---|
| 128 | +0.05% | +2.21% | +7.05% |
| 64 | +0.64% | +1.01% | +5.96% |

Two adapters cost about what one does, within the bands; most of the cost arrives between four and eight. Why it has that shape, these cells do not say. Everything that would answer it — which kernels run, how long, at which batch composition — is below what the campaign recorded, and an explanation from reading the source without a trace to test it against would be a guess written as a finding. It is the next measurement, not this one's conclusion.

## Net of idle, and why that decided the outcome

A cell's window is fixed — 165 s at concurrency 128 — and its 336 requests are in flight for 60 to 65 s of it. The rest is the GPU idling at about 57 W, energy that depends on the window and not on how many adapters the requests use. Divided by the tokens, it dilutes any difference.

So the protocol decides on energy net of idle: the window's energy minus idle power over the time no request was in flight, divided by the tokens. Idle power is measured three times in the same session, at the start, the middle and the end: 57.08, 57.61 and 56.54 W.

The choice is not neutral, and it was made knowing that. Judged on raw energy per token, the same cells give +2.90% at 128 and +2.40% at 64, and H1 would be falsified at both concurrencies. Time per token gives +6.11% and +6.05%. The second dry run had already shown the same split — +7.42% net against +2.77% raw — and the rule was fixed in the amendments with those numbers in front of it, before the campaign's first cell. A rule chosen after seeing the campaign's numbers could have been chosen to produce either answer.

## The imbalance doesn't

d is the skewed configuration against the uniform one at the same N, energy per token net of idle, falsified where |d| is smaller than the larger of the two bands:

| N | Concurrency | d | Bands uniform, skewed |
|---|---|---|---|
| 2 | 128 | +0.52% | 0.60%, 0.73% |
| 2 | 64 | +0.11% | 0.95%, 1.75% |
| 4 | 128 | +0.60% | 0.72%, 0.61% |
| 4 | 64 | +0.46% | 1.02%, 2.01% |
| 8 | 128 | +0.32% | 0.67%, 0.43% |
| 8 | 64 | +0.13% | 0.62%, 0.51% |

Every difference is inside the band. H2 is falsified in all six judgements, each made on its own.

The protocol said H2 would be read first in the tails of the inter-token latency, because a cost that shows up as occasional stalls would vanish in a per-request mean. At p99, skewed against uniform differs by -0.10% to +0.13%, inside both bands at every level. At p90 and concurrency 128 it is higher by +1.35%, +1.29% and +1.11% at N=2, 4 and 8, larger than the bands, which are at most 0.41% — under 0.8 ms on a p90 of 56 to 62 ms. No rule was fixed for latency before the campaign, so these are observations, not judgements, and I report them because the protocol promised to look there.

## The first cell

One cell stands out. `r1-n1u-c128`, the first measured, had a mean time to first token of 3209.5 ms against 2436.1 to 2447.4 ms in the other three repetitions of the same configuration. Its time per output token differs far less, 73.369 ms against 72.467 to 72.633 ms. Its time per generated token — the benchmark's duration over the tokens — is still 4.70% above the mean of the other three, which makes that configuration's band 4.79%, where no other configuration's exceeds 2.12%. The files do not establish why.

The rules keep it, and every judgement above includes it. Left out — an analysis chosen after seeing the data, reported only to show its weight — the H1 margin at 128 moves from +7.05% to +7.33%, and no judgement changes.

## What this does and does not license

On this configuration, the per-adapter counts vLLM computes would not have told two pods with the same adapters apart by cost per token. What moved the cost was how many adapters were served, not how the requests were spread among them. A router that knows which adapters are active on an endpoint already has the variable that mattered here; the one it is missing did not matter.

What it does not license:

- One A10, one base model, eight adapters of one rank and one set of target modules, vLLM 0.30.0, one node session.
- Equal generative work. Every request is 256 in and 256 out. Tenants whose traffic differs in shape — one adapter answering in 30 tokens, another in 800 — are a different experiment.
- Imbalance in volume, not in time. Round-robin interleaves the favoured adapter's requests through the cell; one tenant sending bursts while another trickles is not what was measured.
- Two concurrencies, four repetitions of each configuration.
- The GPU's energy, not the host's.
- Cost per token is not the only reason a router might want the counts. Fairness between tenants, or queueing for one adapter, could still need them; this measures cost.

## Reproducibility

Everything is in [lora-multitenancy-experiment](https://github.com/MicheleCampi/lora-multitenancy-experiment): the protocol with its dated amendments, the harness that ran each cell and checked it was contained in its window, and the files of all 56 cells in `evidence/campaign-2026-09-30/`, with the benchmark's per-request output stripped only of the generated text. No GPU is needed to check the numbers:

    python3 harness/analyze.py evidence/campaign-2026-09-30/results

prints the idle power, the verdict on every cell, cost per token by configuration and both judgements;

    python3 harness/latency.py evidence/campaign-2026-09-30/results

prints the latency figures, after checking each cell against the benchmark's own mean, median and p99. `RESULTS.md` gives the judgements and what they rest on.
