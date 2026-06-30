---
title: About
layout: page
permalink: /about/
---
I'm Michele Campi. I design and build AI infrastructure end-to-end — Rust Kubernetes operators, IaC→GitOps platforms on GKE, observability — with depth in LLM inference (profiling, serving, energy). Everything proven end-to-end on real GPUs, reproducible and documented.

I came to this from an unusual direction. I spent nine years in industrial operations in Italy — production scheduling, cost analysis, margin per machine-hour, the unglamorous economics of how mid-market companies actually make money. That work was quantitative and systems-minded, but it wasn't software. Over the last two years I closed that gap: I learned to build the systems, not just analyze them.

What I build now sits across the AI infrastructure stack — provisioning, orchestration, observability — with LLM inference as the domain I've gone deepest in. I'm drawn to the same question across every project: what is *actually* happening under load, as opposed to what the dashboard says? My profiler, **inferscope**, exists because the gap between client-side latency and what the engine is really doing on the GPU is where inference problems hide. The cold-start work traces that question down to the kernel: an eBPF probe showing that disk I/O is only ~7% of a vLLM cold start, the rest GPU warmup — then a Kubernetes operator that acts on it, marking a service Ready only when it's genuinely warm. The GKE capstone provisions the whole stack as code and proves it on real hardware.

I trace behaviour to the source. When I found NVIDIA Dynamo's KV-router sheds requests under saturation, I read the release-tagged source to confirm the mechanism rather than guess from the metrics. That instinct — measure, don't assume; read the code, don't trust the abstraction — is the throughline.

I also keep a production service running, **OptimEngine**: an OR-Tools optimisation service exposed over REST and MCP, with full OpenTelemetry tracing and a public Grafana dashboard. It's where the operations-domain background and the engineering meet.

I write about this work at roughly one article a month — inference performance, observability, the things measurement reveals that intuition misses.

**Where to find me:** github.com/MicheleCampi

I work as a builder: I design and validate systems end-to-end — AI-assisted, in long async focus blocks — rather than firefight them. I'm open to remote, async infrastructure roles: platform engineering, Kubernetes, observability, with depth in AI/LLM inference. If you're hiring at that layer, I'd like to hear from you.
