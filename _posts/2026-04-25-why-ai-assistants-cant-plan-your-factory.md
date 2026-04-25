---
title: "Why your AI assistant can't actually plan your factory: a real test with a German SME manufacturer"
date: 2026-04-25
tags: [optimization, manufacturing, industry-4-0, or-tools, ai-agents, scheduling]
---

# Why your AI assistant can't actually plan your factory: a real test with a German SME manufacturer

Last week I sat down with a synthetic but realistic problem. **MetallbauTech GmbH**, a 45-person precision manufacturer in the Stuttgart area, has fifteen CNC machines and six automotive orders to deliver this week. Mix of brake calipers for BMW, transmission gears for Porsche (with a yield-rate constraint, because precision), Audi steering parts, a Tesla rush order due in 15 hours, MAN truck components, and a standard machining batch.

A Monday-morning question for the production manager: what's the optimal weekly plan?

I wanted to test two things, both relevant to anyone running an SME factory in 2026:

1. **Can a mathematical solver actually do better than spreadsheets and intuition?** (Spoiler: yes, dramatically.)
2. **Can a generic AI assistant — like ChatGPT or Claude — do the same job?** (Spoiler: no, and not for the reasons you'd expect.)

This post walks through the actual tests, with real outputs and real timings. If you run a factory with 10–50 machines and you're tired of Excel-Gantt week planning, the patterns here will be familiar. If you're evaluating "AI for manufacturing" tools, you'll see exactly where the line is between marketing and reality.

## The problem, in plain numbers

Here's MetallbauTech's week:

- **15 machines**: 8 CNC milling stations (two of them on night shift, available 80h instead of 45h), 4 CNC lathes, 2 precision grinders, 1 quality measurement station
- **6 orders**: priorities 3 to 10, due dates ranging 15h to 42h, total 19 production tasks across all orders
- **Constraints**: each task can only run on certain machines (you can't do precision grinding on a lathe), Porsche gears require a machine with ≥98% yield rate, the QC station bottlenecks at the end
- **Objective**: minimize total tardiness — the sum of how late each order is past its due date

A human production manager — and I've watched several do this — typically takes half a day to lay out a plan like this on a whiteboard or in a spreadsheet, and the result is rarely optimal. They prioritize by gut, parallelize where they remember they can, and accept that some orders slip.

This is exactly the kind of problem **Flexible Job Shop Scheduling (FJSP)** was designed to solve mathematically. Google's OR-Tools CP-SAT solver, wrapped in a service layer I built called **OptimEngine**, eats this for breakfast.

## Test 1: deterministic schedule

I wrote the scenario as JSON — 6 jobs with their tasks, 15 machines with their availability and yield rates, the optimization objective — and POSTed it to OptimEngine's `/schedule-robust` endpoint. This is a composite endpoint that, given a scheduling problem, returns the optimal plan.

```bash
curl -X POST https://optim-core-gateway-production.up.railway.app/schedule-robust \
  -H "X-Core-Key: ***" \
  -H "Content-Type: application/json" \
  -d @metallbautech-scenario.json
```

The response came back in **1.28 seconds**. Solver status: `optimal` (not heuristic — provably optimal under the constraints). Result:

- **Makespan: 18 hours.** The entire week's production fits in 18 hours of wall-clock time.
- **6 of 6 orders on time.** Zero tardiness. Including the Tesla URGENT order (due in 15h), which the solver completed in 5h flat.
- **Plan structure**: Tesla used CNC-M-06 for rough milling and CNC-M-01 for finishing — not the "obvious" CNC-M-01 first. The solver figured out that running Tesla and BMW in parallel on different machines was faster than serializing them. Porsche correctly went to GRIND-01 (yield 0.99, satisfying the ≥0.98 constraint).

The schedule wasn't intuitive. A production manager working manually would likely have started Tesla on the "best" machine and serialized everything else around it. The solver found a parallel plan where Tesla finishes in 5h on the second-tier mill while the first-tier mill handles BMW concurrently.

This alone is the value proposition: **a solver finds non-obvious optimal plans in seconds**. But there's a catch, and it's the reason most schedule outputs don't survive contact with reality.

## Test 2: what happens when reality bites

The 18-hour plan above assumes all task durations are exact. They aren't. In a real precision shop, the gear-cutting operation on Porsche transmissions might take 5 hours on a good day, 7 nominally, 10 if a tool needs replacement. The Tesla rough milling might run 1.5h to 4h depending on material variability and operator experience.

If you give your production manager an "18 hour optimal plan" and don't tell them the underlying assumptions, they'll commit to it. Then on Wednesday, when Porsche's gear-cutting runs over, the cascading delays push Tesla off its 15h deadline. By Thursday afternoon, you're calling the customer.

OptimEngine's `/schedule-robust` endpoint accepts an optional `stochastic_parameters` array — a way to declare which parameters are uncertain and what distribution they follow. I added three:

- Tesla rough-milling: triangular distribution, min 1.5h, mode 2.5h, max 4h
- Porsche gear-cutting: triangular, min 5h, mode 7h, max 10h  
- Porsche grinding: triangular, min 3h, mode 5h, max 8h

Re-ran the call. Total time: **1.83 seconds** (147ms scheduling + 793ms running 30 Monte Carlo scenarios). The response now contained three strategies:

**Strategy A — Nominal Optimistic**: same 18-hour plan as before. Risk level: high.

**Strategy B — CVaR-Protected**: makespan 28 hours under the 95% CVaR worst case. Expected value 23 hours. Feasibility rate 100%. The recommendation field reads:

> "High variability: CV=15.5%. The makespan ranges from 18.0 to 29.0. Risk-aware planning strongly recommended. The gap between expected value (23.0) and CVaR 95% (28.0) is 21.7%. This is the 'risk premium' — what you pay for 95% protection."

**Strategy C — Balanced**: middle ground. Use the nominal schedule but add buffer time on the parameters identified as uncertain.

This is the part that matters for a CEO making delivery commitments. The nominal plan said 18 hours. The probabilistic analysis says: **the realistic expected outcome is 23 hours, and there's a 5% chance you blow past 28 hours**. If you committed to the customer based on Strategy A, you're now exposed.

The cost of running this analysis is **0.79 seconds of computation**. The cost of not running it is the phone call to BMW on Thursday.

## And what about asking ChatGPT or Claude?

Here's where it gets interesting. While testing OptimEngine, I asked Claude (in a separate conversation, no MCP integration) to plan the same MetallbauTech week using natural language. The query was straightforward:

> "I work at MetallbauTech GmbH, a German manufacturer with 15 CNC machines. I have 6 automotive orders this week (BMW brake calipers, Porsche transmission gears, Audi steering, standard batch, Tesla motor mounts URGENT due in 15h, MAN truck parts). Each order has multiple machining stages. Can you build me an optimized weekly schedule that minimizes total tardiness?"

Claude responded thoughtfully. It flagged its assumptions explicitly (synthetic operation data, generic machine pool), built an FJSP model, then ran CP-SAT in its sandbox. **It produced a schedule in 0.06 seconds.** It even generated a Gantt chart.

Impressive. Until I asked the follow-up: "Did you call the OptimEngine MCP server I have configured, or did you run CP-SAT locally?"

The honest answer:

> "I ran CP-SAT locally in my sandbox. I pip install'd ortools and called the Python solver directly — I never called your OptimEngine MCP server. [...] My local model is a reasonable FJSP with weighted tardiness, but I have no guarantee it matches OptimEngine's actual /optimize_schedule contract — field names, objective options, how alternatives are encoded, whether setup_times and availability_windows are passed the way I'd assume from memory. The solution is mathematically valid for the model I built; it is not a verified OptimEngine output."

This is the line that matters. **A generic AI assistant can improvise CP-SAT for a small problem and produce something plausible. But "plausible" is not "production-grade."**

Here's what the AI's local reconstruction was missing, and what your factory will care about:

1. **Reproducibility.** The AI's solver runs in a sandbox that's recreated each session. No two runs are identical environments. Production scheduling needs runs that are byte-identical given the same inputs, for audit and rollback.

2. **Custom logic.** OptimEngine's v9 solver has nine years of hand-tuning for manufacturing scenarios — sequence-dependent setup times, multi-window availability, yield-rate filtering, four optimization objectives, four uncertainty modes. The AI improvised "a reasonable FJSP" — generic, but missing every detail that makes a real shop's plans actually executable.

3. **Performance at scale.** AI sandboxes run for one conversation. They don't run 24/7 with SLAs, can't be called by automated agents at 1000 requests per hour, don't return responses in <2 seconds under load. Your factory's MES integration needs an actual API endpoint with uptime guarantees, not a chat tab.

4. **Specialization beyond scheduling.** OptimEngine exposes ten composite endpoints — not just `/schedule-robust`, but `/risk-analysis`, `/full-intel`, `/pack-resources`, `/forecast-basic`, etc. Each is pre-orchestrated for a specific class of decision. A generic AI would have to invent the orchestration each time, with no memory of last week's decisions, no understanding of your specific shop's bottlenecks.

5. **Auditability.** When the auditor asks why you made a specific scheduling choice in February, "because I asked Claude" doesn't pass ISO 9001. "Because OptimEngine `/schedule-robust` v9.0.2 returned this strategy with these parameters and these inputs, logged with timestamp, signature, and CVaR analysis" does.

The AI assistant was honest about this. Most won't be. The current generation of "AI for X" tools either avoids these questions or hides behind vague capability claims. **Your AI assistant can sketch a Gantt chart. It cannot run your factory.**

## What this means if you're an SME manufacturer

If you have 10–50 machines and you're juggling multi-client orders weekly, three things follow from this:

**One: a real solver is now within reach.** You don't need an SAP/Siemens enterprise contract anymore. OptimEngine is an HTTP API — anyone with a backend developer can integrate it in a day. Send your jobs and machines, get back an optimal plan with risk analysis. That's the whole story.

**Two: probabilistic planning is a competitive advantage.** Every competitor still using deterministic Excel plans is silently exposed to the variance you're now measuring. The 21.7% risk premium between nominal and CVaR-95 is the gap they don't see. You will.

**Three: AI assistants are not the answer for production decisions, but they're a great front-end.** A natural-language interface that converts a CTO's question — *"can we squeeze in one more order this week?"* — into an OptimEngine call, runs the solver, and returns the answer in business terms is exactly the right architecture. The AI handles the conversation; the solver handles the math.

## Try it

If your factory's weekly planning sounds like the MetallbauTech scenario above, OptimEngine's `/schedule-robust` endpoint is live. The full request schema is in the [public OptimEngine documentation](https://optim-core-gateway-production.up.railway.app/) (open to inspection — no signup needed to read).

The 6-job, 15-machine, 19-task scenario from this article runs in under 2 seconds and returns three strategies. Your real shop floor — likely 30–80 jobs across 10–50 machines — runs in 5–30 seconds.

If you're an SME manufacturer in Italy, Germany, or Europe more broadly, and you'd like to discuss a pilot integration — connecting OptimEngine to your existing MES, ERP, or planning workflow — reach out. I work with manufacturing operations specifically, with a controlling background in contract manufacturing before transitioning to engineering.

The math is solved. The integration is the part that matters.

---

*OptimEngine is a mathematical optimization service built on Google OR-Tools CP-SAT (v9.0.0), exposing 11 solver capabilities including FJSP scheduling, CVRPTW routing, bin packing, Pareto multi-objective analysis, Monte Carlo risk simulation with CVaR metrics, parametric sensitivity analysis, and prescriptive intelligence. The engine is currently deployed for industrial use cases and accessible via REST and (forthcoming, OAuth-gated) MCP protocols.*
