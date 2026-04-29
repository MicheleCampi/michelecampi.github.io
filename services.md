---
title: Services
layout: page
permalink: /services/
---

## Controller turned builder

Bridging operations expertise with frontier optimization tools — for manufacturers who suspect their planning is leaving money on the floor.

Most mid-market manufacturers have no shortage of operational data. They have shortages of people who can read the data, model the system underneath it, and translate the math into decisions a CFO can sign off on. That's the work I do.

Below are four shapes that work usually takes. None of them require committing to a long engagement before there's something concrete on the table.

---

### Operational audit + quantitative analysis

A focused assessment of a scheduling, planning, capacity, or risk problem inside your plant. I model the system, run the analysis on real or representative data, and return a written report with findings, modeled alternatives, and ROI estimates.

The deliverable is the report — not slides, not a sales pitch. You read it, share it internally, decide what to do next.

*Format: 1-3 weeks · Written deliverable · Remote or on-site*
*Best for: mid-market manufacturers (€5-50M revenue) with chronic operational frustration nobody has quantified*

---

### Working prototypes and proofs of concept

Building a functioning prototype that demonstrates whether an idea is viable, before committing to a full implementation. The prototype runs on your data (or representative data), produces real outputs, and answers the question "would this actually work?" with evidence rather than opinion.

Often the conclusion is "yes, build the production version." Sometimes it's "no, the constraints kill the value." Both outcomes are valuable.

*Format: 2-6 weeks · Working software · Documented handover*
*Best for: companies considering a custom internal tool or evaluating a vendor's claims*

---

### Technical roadmap and architecture review

A written document that helps non-technical leadership evaluate a digitalization project. Includes proposed architecture, step-by-step roadmap, cost and effort estimates, integration considerations, and risk assessment.

The deliverable is what your team needs to make a confident build-or-buy decision, request budget approval, or set scope with an external implementer.

*Format: 1-2 weeks · Written deliverable · Vendor-neutral*
*Best for: companies starting an operations digitalization initiative and wanting independent input before committing*

---

### Bridge work between business and technical teams

Ongoing presence in projects where the business knowledge sits with operations or controlling, and the technical execution sits with internal IT or external consultants — and the two groups are talking past each other.

I translate. I keep technical scope honest about business reality. I write the documentation that connects the two worlds. I sit in the meetings nobody wants to sit in but everybody needs.

*Format: monthly retainer or project-based · Hybrid remote/on-site*
*Best for: digitalization projects that have stalled or are heading toward expensive misalignment*

---

## What's underneath the work

I built and operate **OptimEngine**, a production-grade mathematical optimization service exposed via REST API and dual-stack MCP (open SSE + OAuth 2.1-gated Streamable HTTP). It runs on Google's OR-Tools CP-SAT solver and handles flexible job-shop scheduling, vehicle routing with time windows, bin packing, Pareto multi-objective analysis, Monte Carlo risk simulation with CVaR metrics, and parametric sensitivity analysis.

It's live, it's deployed, it returns optimal schedules in milliseconds on realistic problems, and it's the tooling I use when client work calls for the same kind of math.

Concretely:

- **11 mathematical solvers** covering the most common operations decision problems
- **Optimal schedules in 0.04–2 seconds** on realistic mid-market scenarios
- **Stochastic / robust optimization** for planning under uncertainty
- **REST API + MCP interfaces** — usable by humans and AI agents alike

OptimEngine is open visibility on [GitHub](https://github.com/MicheleCampi) and reachable through the [x402 payment gateway on Arc](https://optim-arc-v3.vercel.app) for autonomous AI agent integrations.

---

## Recent writing

A few articles relevant to the kind of work above:

- **[What an OR-Tools solver finds in a week of contract packaging](https://michelecampi.github.io/2026/04/29/or-tools-week-contract-packaging.html)** — A realistic mid-market scenario with real solver output. The interesting finding isn't the 15% scheduling gain, it's what the solver shows about hidden capacity.

- **[Three production scheduling failures I've seen, and the math that would have caught them](https://michelecampi.github.io/2026/04/26/three-scheduling-failures-and-the-math-that-would-have-caught-them.html)** — Three operational patterns that repeat across European mid-market plants, each with a hidden cause and a quantitative method that would have caught it.

- **[Why your AI assistant can't actually plan your factory](https://michelecampi.github.io/2026/04/25/why-ai-assistants-cant-plan-your-factory.html)** — A direct test comparing a frontier AI assistant to a constraint programming solver on a realistic scheduling problem. The gap is structural.

The full archive lives on the [blog](https://michelecampi.github.io/).

---

## How I work

A few things worth being explicit about before any conversation starts.

**Engagement size.** Most useful starting points are scoped between €3,000 and €15,000 — small enough to be a low-risk decision for a CFO, large enough to produce something concrete. Larger and longer engagements are possible after a first piece of work has demonstrated mutual fit.

**Geographic focus.** Italian and broader European mid-market is the primary focus. Italy, Germany, Austria, Switzerland in particular. Open to remote engagements globally — most of the work is async-friendly by design.

**What I don't do.** I'm not an APS vendor, an ERP integrator, or a generic AI consultant. The work I take on has quantitative output as the deliverable — numbers, models, working code, written analysis. Strategy decks are not my primary product.

**How a first conversation usually goes.** Email is the channel. Describe the situation in 5-10 lines — what's broken, what you've tried, what good would look like. I respond with whether the problem fits what I do, an honest read of how big the engagement would need to be, and what the next concrete step looks like.

---

## Contact

The fastest way to start a conversation is by email. I read everything and respond within a few business days.

📧 **michele.campi [at] outlook.com** *(replace [at] with @)*

You can also reach me via [GitHub](https://github.com/MicheleCampi) or [X](https://x.com/MicheleC54474).
