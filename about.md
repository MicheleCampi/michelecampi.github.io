---
title: About
layout: page
permalink: /about/
---
I'm Michele Campi. I build the platforms that serve LLM inference — Kubernetes operators in Rust, Terraform→GitOps on GKE and AWS EKS, observability — so that a GPU fleet places, recovers and reports on itself rather than being driven by hand. The inference layer is where I've gone deepest: profiling, serving, energy, and what a workload actually costs.

I came to this from an unusual direction. I spent nine years in industrial operations in Italy — production scheduling, cost analysis, margin per machine-hour, the unglamorous economics of how mid-market companies actually make money. That work was quantitative and systems-minded, but it wasn't software. Over the last two years I closed that gap: I learned to build the systems, not just analyse them.

## The question I keep asking

What is actually happening under load, as opposed to what the dashboard says?

My profiler, [inferscope](https://github.com/MicheleCampi/inferscope), exists because the gap between client-side latency and what the engine is really doing on the GPU is where inference problems hide. The cold-start work traces that question down to the kernel: an eBPF probe showing disk I/O is only ~7% of a vLLM cold start, the rest GPU warmup — then [a Kubernetes operator](https://github.com/MicheleCampi/vllm-coldstart-operator) that acts on it, marking a service Ready only when it is genuinely warm, and grown since into a fleet orchestrator that survives spot preemption make-before-break. The GKE capstone provisions the whole stack as code and proves it on real hardware, with a public EKS twin showing the same GitOps contract on AWS.

More recently the question turned to money. An agentic workload leaves the GPU allocated and idle while it waits on tools, and [I measured what that costs](/observability/systems-engineering/llm-inference/2026/08/06/agentic-trajectory-cost.html): the cost of generating holds flat within a 0.5% band while the cost of waiting grows 25×, so the entire increase across the sweep is allocation rather than work. The obvious platform response — release the GPU during tool calls — repays on none of the fifteen cells measured, because getting the GPU back costs more than any tool call in the published range.

Those two axes plus a packing bound — how many concurrent trajectories one replica actually holds, derived from the cost data and then tested on hardware rather than shipped — are capacity planning for agent infrastructure, measured instead of modelled. The limit is stated with the result: in the bound test the trajectories ran in lockstep and shared their idle time rather than interleaving, so the figure holds for synchronised replicas and the staggered case is still open.

The most recent one closed a question I had kept open across three campaigns, and the answer was no. An efficiency-aware placement strategy — put the replica where the node runs more efficiently — beat the simpler alternative by 2.05%, with a confidence interval straddling zero and a threshold of 3% fixed before the run. A negative result, published as one. What made it a real answer rather than another artefact of my own instruments is that the strategies diverged the way they were meant to; the July attempt had produced a similar number for a reason that had nothing to do with the strategy.

## How I work, which matters more than what I've built

I trace behaviour to the source. When I found NVIDIA Dynamo's KV-router sheds requests under saturation, I read the release-tagged source to confirm the mechanism rather than guess from the metrics. When a config for someone else's codebase looked correct, I deleted the block it exists to provide to see whether my own test would notice — it didn't, which told me something about the system I would not otherwise have known.

That habit is the throughline, and I [wrote it up](/observability/systems-engineering/llm-inference/2026/08/16/checks-that-found-them.html) with the six defects it caught on one campaign: a figure that was right until a flag made it wrong, a dry-run blind at exactly the value that mattered, a rounding made against my own result in the mistaken belief that understating is the cautious direction. Three of the six were in the checking rather than in the work.

Two things follow from it in practice. Experiments carry a falsification criterion stated before the run, so the reading cannot be chosen after seeing the number — and when a result comes back negative, that is what gets published. And every figure I claim regenerates from committed evidence with a command, because a number without a way to reproduce it is a number a reader has to take on faith.

## Upstream

Measuring your own code is the easy half. I contribute to the projects this work sits on: merged into [llm-d](https://github.com/llm-d/llm-d-router/pull/2037) — removing a mid-stream latency-prediction path from the inference scheduler, on an issue a maintainer had assigned to me — as well as NVIDIA AIPerf and mistral.rs. Working inside a codebase you don't own, to the standard its maintainers require, is a different skill from building your own, and the only way to demonstrate it is to do it.

I also keep a production service running, [OptimEngine](https://optim-engine-production.up.railway.app): an OR-Tools optimisation service exposed over REST and MCP, with OpenTelemetry tracing and a public Grafana dashboard. It is where the operations background and the engineering meet.

## Writing

Eighteen articles since April 2026, roughly four a month — inference performance, observability, and the things measurement reveals that intuition misses. The ones I'd start with are the [trajectory cost study](/observability/systems-engineering/llm-inference/2026/08/06/agentic-trajectory-cost.html) and the [methodology post](/observability/systems-engineering/llm-inference/2026/08/16/checks-that-found-them.html).

## Working together

I work as a builder: I design and validate systems end-to-end, AI-assisted, in long async focus blocks — rather than firefight them. The reliability work I do is aimed at removing the firefighting, not at staffing it.

I'm open to remote, async-first platform roles: Kubernetes, Terraform/GitOps, observability, with depth in AI and LLM inference. If you're hiring at that layer, I'd like to hear from you.

Where to find me: [github.com/MicheleCampi](https://github.com/MicheleCampi)
