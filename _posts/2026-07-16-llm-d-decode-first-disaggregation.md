---
layout: post
title: "Disaggregation isn't a deployment topology. In llm-d, it's a per-request decision."
date: 2026-07-16
categories: systems-engineering llm-inference
tags: kubernetes llm-d disaggregation prefill decode epp gateway-api routing vllm kv-cache
---

Prefill/decode disaggregation has a reputation problem: it gets discussed as if it were a cluster-level switch. You either run disaggregated or you don't. Both answers are wrong, and I have measurements for both directions.

Disaggregating *always* is wrong because the KV-cache transfer between prefill and decode workers is not free. In [my experiment with NVIDIA Dynamo's disaggregated serving](/observability/systems-engineering/llm-inference/2026/06/27/dynamo-kv-router-saturation.html), the NIXL KV transfer showed up directly as a TTFT cost against an aggregated baseline on the same hardware. For short prompts, you pay the transfer and get little back.

Disaggregating *never* is also wrong, because prefill and decode are wildly different workloads pretending to share a GPU. In the same experiment, under growing decode load, the prefill worker idled down to ~127W while the decode worker peaked at 466W — a 3.7× power asymmetry between two identical GPUs, against ~1.0× for the aggregated pair. Compute-bound and memory-bound phases interleaved on one worker interfere with each other precisely when load is highest.

If both static answers are wrong, the decision has to move down a level: not "is this cluster disaggregated" but "should *this request* be disaggregated". That is exactly where llm-d puts it, and this post is a read-through of how the scheduler actually makes that call — from the source of `llm-d/llm-d-router`, where the EPP (endpoint picker) code now lives after migrating out of `kubernetes-sigs/gateway-api-inference-extension`.

## One pool, not two

The first thing the architecture gets right is what it *doesn't* do: there are no separate InferencePools for prefill and decode. There is a single InferencePool, workers carry a `llm-d.ai/role` label, and the EPP distinguishes roles through scheduling profiles. The routing contract stays singular — the gateway talks to one pool — and role topology becomes a scheduling concern rather than an API-surface concern. Whether a request runs aggregated or disaggregated is invisible above the EPP.

## Decode-first

Inside the EPP, the profile handler orchestrates the decision iteratively: it does not plan the whole request path upfront. First it runs the pick for the **decode** worker. Only then does the disaggregation decider run — and it runs *seeing the decode worker that was already selected*.

This ordering looks like an implementation detail. It is the whole point.

## The prefix-based decider

The decider that ships for the P/D case is prefix-based. Given the selected decode worker, it estimates how many prompt tokens are *not* already covered by that worker's prefix cache: the cached blocks (unweighted) times the block size, subtracted from the prompt. If `nonCachedTokens` stays under a threshold, the request goes decode-only — aggregated, no remote prefill, no KV transfer. Only when the uncached portion is large enough does the handler pick a prefill worker and split the request.

And this is why decode-first is load-bearing: prefix-cache state is per-worker. The question "is this prompt already cached?" has no answer until you know *which* decode worker you're asking about. A decider that ran before decode selection would be reasoning about a cache that doesn't exist yet. The iterative Pick isn't a convenience — it's what makes the cache-aware decision well-posed.

The failure posture is equally telling: if the decider can't produce a decision, the fail-safe is decode-only. When in doubt, don't disaggregate. Given that the cost of a wrong "disaggregate" is a wasted KV transfer on the critical path and the cost of a wrong "don't" is some redundant prefill compute, that default is the right asymmetry.

## Recovery is asymmetric too

The same asymmetry shows up in error handling, and it's my favorite design signal in the codebase. If the **decode** worker fails, the request fails — there is nothing to degrade to, decode *is* the request. If the **prefill** worker fails, the request does not fail: it silently degrades to aggregated, with the decode worker doing its own prefill.

Read as a design statement: remote prefill is an optimization; decode is the contract. The system treats disaggregation the way a good cache is treated — something whose absence costs performance, never correctness.

## Where this is heading

The P/D handler I read is itself being generalized: there is active work upstream toward a handler with three roles — prefill, decode, and a third encode role (P/D/E). The per-request, decode-anchored structure carries over; the role set widens. I'm flagging it as work-in-progress rather than describing it as settled, because it isn't.

## What I did with this

Reading this code reshaped the orchestration design of my own operator. [ADR-0006 in `vllm-coldstart-operator`](https://github.com/MicheleCampi/vllm-coldstart-operator/blob/main/docs/adr/0006-disaggregation-aware-orchestration.md) adopts the same posture at the fleet level: roles as a refinement of a single `FleetService` rather than separate services, warmth semantics that are deliberately asymmetric per role, and recovery that is role-differentiated in exactly the sense above — a lost prefill worker degrades capacity, a lost decode worker is an incident. The motivating economics are real: cache-regime shifts alone moved energy efficiency by 69.2% tokens/joule in my measurements (detailed write-up coming in August), and role-aware placement is how you defend the good regime.

## Takeaway

Disaggregation in llm-d is not a topology you deploy. It's a decision the scheduler makes per request, anchored on the decode worker, gated by that worker's actual cache state, and designed to fail *toward* aggregated serving rather than away from it. If you're evaluating disaggregated inference, the question to ask your stack is not "does it support P/D" but "who decides, when, and what happens when prefill dies".

Links: [the Dynamo experiment](/observability/systems-engineering/llm-inference/2026/06/27/dynamo-kv-router-saturation.html) · [`llm-d/llm-d-router`](https://github.com/llm-d/llm-d-router) · [ADR-0006](https://github.com/MicheleCampi/vllm-coldstart-operator/blob/main/docs/adr/0006-disaggregation-aware-orchestration.md)
