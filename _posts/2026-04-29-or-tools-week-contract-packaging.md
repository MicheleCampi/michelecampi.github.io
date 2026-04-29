---
title: "What an OR-Tools solver finds in a week of contract packaging — and what the planner usually misses"
author: Michele Campi
published: true
description: "A synthetic but realistic case study on a mid-market contract packager: eight customer orders, six production lines, sequence-dependent setup times. The expert manual schedule lands around 190 quarter-hours of makespan. OptimEngine returns the proven optimum in ten milliseconds: 161 quarters. The interesting finding isn't the 15% gain — it's what the solver shows about hidden capacity that the manual planner can't see."
date: 2026-04-29
tags: [or-tools, scheduling, manufacturing, optimization, contract-packaging]
---

# What an OR-Tools solver finds in a week of contract packaging — and what the planner usually misses

Most arguments in favor of optimization software focus on the obvious benefit: a solver finds a better schedule than a human planner, faster. That part is true and uninteresting. The interesting part is what the solver shows you about your own operation that you couldn't see before.

This article walks through one synthetic but realistic week at *Lombarda Confezionamenti SRL*, a fictional contract packager in northern Italy. The company is fictional; the operational pattern is one I've seen repeat across European mid-market contract packagers in seven years of operations work. The numerical input is deliberately ordinary. The numerical output — the schedule, the metrics, the bottleneck analysis — is what OptimEngine actually returned when I fed the input into the solver. No invented numbers.

The takeaway isn't "automate scheduling." The takeaway is closer to: *your manual planner is doing the visible part of the job correctly, but the spreadsheet hides what the solver makes obvious — that four of your six lines are running half-empty most of the time.*

## The setup

Lombarda Confezionamenti is a contract packager for personal care brands: shampoos, body washes, creams, lotions, fragrances. €18M revenue, ~85 employees, two-shift operation (16 hours per day, five days per week). Six production lines, each specialized in different parts of the packaging flow:

- **L1** — Heavy filling line (200-500ml bottles)
- **L2** — Medium filling line (50-150ml jars and tubes)
- **L3** — Automatic cartoning line
- **L4** — Labelling line
- **L5** — Bundling and multipack line
- **L6** — QC station with batch release

Format changeovers on these lines aren't trivial. On the heavy filling line, a switch between products requires roughly two hours of cleaning, machine adjustment, and validation. On the medium filling line, it's around 90 minutes. On cartoning and labelling, an hour each. Bundling is the cheapest at 30 minutes. QC has no setup. These are real numbers from this kind of plant.

The week in question has eight orders from five different customers. Each order requires a different sequence of operations, depending on the product. Here's the load:

| Order | Customer | Product | Sequence | Volume |
|-------|----------|---------|----------|--------|
| J1 | Customer A | Face cream 50ml | filling → cartoning → labelling → QC | 8,000 units (rush, 48h deadline) |
| J2 | Customer B | Shampoo 250ml | filling → labelling → QC | 12,000 units |
| J3 | Customer C | Detergent 500ml | filling → labelling → bundling → QC | 6,000 units |
| J4 | Customer A | Body wash 200ml | filling → labelling → QC | 10,000 units |
| J5 | Customer D | Body lotion 150ml | filling → cartoning → QC | 4,000 units |
| J6 | Customer B | Conditioner 250ml | filling → labelling → bundling → QC | 8,000 units |
| J7 | Customer E | Perfume 100ml | cartoning → QC | 3,000 units (no filling) |
| J8 | Customer C | Liquid soap 300ml | filling → labelling → QC | 7,000 units |

Total processing time across all operations, ignoring setups: roughly 79 hours of machine work distributed across six lines. With perfect parallelization and zero setup, the absolute lower bound on makespan would be around 13 hours. Reality is much further from that.

## What the experienced planner does

On Monday morning at 7am, the production manager opens the Excel file. He has done this for fifteen years. He reads the rush order from Customer A, marks it as priority one. Customer A is a strategic account — they also have order J4 — so he pencils in J4 second. Then he groups the remaining orders by customer to minimize "context switching" mentally: Customer B (J2 and J6 together), Customer C (J3 and J8 together), then D and E.

Within each block, he assigns tasks to lines using rules he doesn't articulate but follows consistently:

- Long fillings on L1 because that's the heavy line
- Small fillings on L2
- One job at a time on the bottleneck line, mostly — this is the heuristic he trusts most
- Setup planned at the start of each job, never overlapped with anything

He blocks out the week. The schedule he produces, when I trace it through the same constraints I gave the solver, terminates around **190 quarter-hours of makespan**. That's about 47.5 hours of plant time, which means he closes the week sometime late Wednesday or early Thursday — roughly three working days, given the two-shift schedule.

This is a defensible, professional schedule. The rush is delivered on time. No customer is forgotten. Setup costs are managed. He's been doing this competently for fifteen years.

## What OptimEngine does

I fed the same inputs — eight jobs with their task sequences, six lines with their setup times, the priority ranking, the rush deadline — into OptimEngine's CP-SAT scheduler.

The solver returned a status of `optimal` in **10 milliseconds**.

The makespan it found: **161 quarter-hours**. That's 40.25 hours, or roughly 2.5 working days at two shifts. About **15% better than the manual schedule**.

Zero orders late. The rush J1 finishes at quarter-hour 57, which is 7 quarters before its 64-quarter deadline — comfortable margin without overcommitting capacity to the rush.

The solver's gain over the manual baseline is not coming from any single brilliant move. It's coming from many small parallelizations the human eye doesn't easily see. At time zero, three things start in parallel: J6 begins filling on L1, J1 begins filling on L2, J7 begins cartoning on L3. The manual planner usually starts L1 first and then thinks about L2 once L1 is "moving." The solver doesn't think; it just maps the constraint graph and moves everything that can move.

Format changeovers on L1 are also handled aggressively. Five different products run on L1 across the week: J6 → J2 → J3 → J4 → J8. Each transition costs eight quarter-hours of setup. The solver sequences them in the order that doesn't force any other line to wait. The manual planner often runs setups during the night shift "to keep the day shift productive," which sounds smart but actually adds idle time elsewhere.

## The interesting finding: average machine utilization is 31.3%

Here's where the article would normally end with "and that's why you should buy optimization software." But the solver returns more than a schedule. It returns metrics. And one of them is uncomfortable:

**Average machine utilization across the week: 31.3%**.

Let me break that down by line:

| Line | Utilization | Tasks | Notes |
|------|-------------|-------|-------|
| L1 (heavy filling) | 69.6% | 5 | Bottleneck |
| L2 (medium filling) | 17.4% | 2 | Underused |
| L3 (cartoning) | 18.6% | 3 | Underused |
| L4 (labelling) | 50.3% | 6 | Secondary bottleneck |
| L5 (bundling) | 11.2% | 2 | Severely underused |
| L6 (QC) | 20.5% | 8 | Underused |

L1 is running roughly 70% of the available time. L4 is at half capacity. The other four lines are sitting idle most of the week. This is true *under the optimal schedule* — there's no sequencing improvement that would change this picture. The reason these lines are underused is structural: the order mix this week happens not to need much medium filling, much cartoning, much bundling, or much QC throughput.

The manual planner can't see this. He sees that the week "got done." He sees that the rush was delivered on time. He doesn't see that L2, L3, L5 sat idle for over 30 hours each. They were never on his dashboard because they weren't constraining his completion date. The bottleneck has all the visibility; the slack has none.

For a plant manager, this is the most actionable insight in the entire schedule. It's not "your planner could be better." It's "this week, you have roughly 100 hours of free capacity on four lines that nobody is selling." Those four lines have an industrial cost — depreciation, energy in standby mode, maintenance contracts, operator availability — that runs whether they're packaging product or not.

For a CFO, the question becomes: *what additional orders, with what setup profile, would absorb the slack on L2, L3, L5, and L6 without overloading L1?* That's a commercial question, not a scheduling question. But the scheduling output is what makes the question even visible.

## What this case shows, and what it doesn't

I want to be careful with what this analysis proves and what it doesn't.

It does prove that, on a realistic week of contract packaging, an OR-Tools solver finds a schedule about 15% shorter than what an experienced human planner would produce in 30 minutes of paper-and-spreadsheet work. That gain is real and consistent across most of the FJSP scheduling problems I've tested. It comes from parallelism that humans don't naturally compute.

It also shows that the solver surfaces structural information — line utilization, bottleneck analysis, idle capacity — that doesn't appear on the planner's Monday-morning whiteboard. This information is more valuable than the 15% scheduling gain in most plants I've worked with, because it points at commercial decisions, not just operational ones.

What the analysis does *not* prove is that automating scheduling alone fixes anything. The 15% gain only matters if the plant can absorb it: if the order book grows, if the planner uses the time saved on something else, if the customer accepts faster delivery. Plenty of plants would just see the 15% as a softer week and do nothing differently. That's not a software problem; that's a management problem.

It also doesn't prove that this particular plant should immediately invest in optimization software. The 15% scheduling gain at this volume is worth, very roughly, €40-60K per year of recovered capacity at typical mid-market industrial costs. That's not life-changing, and it has to be weighed against the cost of integration, training, and the change management work of asking a fifteen-year-veteran planner to trust a black box.

Where I see optimization tools actually pay off in mid-market is two situations:

**First**, when the plant has a real growth ceiling that could be lifted. If management is looking at L1 utilization and thinking "we should buy a second heavy filling line," but L2, L3, L5 are at 17%, the better question is whether sales mix can be rebalanced before capex. The solver makes that question quantitative.

**Second**, when the plant has visible service problems — rushes accepted at high cost, deadlines slipping under load, last-minute customer changes producing chaos. The same solver run with robust optimization extensions can quantify how brittle the current schedule is to disruption, and how much slack would buy how much resilience. That's a different article.

## A note on what's underneath

The solver used in this analysis is OptimEngine, built on Google OR-Tools CP-SAT. The math is mature: constraint programming applied to flexible job-shop scheduling is a well-developed field with decades of research behind it. What's new in 2026 is mostly the accessibility — the same kind of math that enterprise APS vendors charge mid-market companies €100-300K per year to access can now be packaged as a service that fits the operational and financial profile of an Italian PMI.

The endpoint that returned the schedule for this case study is the same one I exposed publicly through MCP and x402 payment infrastructure earlier this month, and that I've been writing about in the rest of this blog. If you're a mid-market manufacturer or contract packager wondering whether your week looks like Lombarda Confezionamenti's, the question I'd start with is the one this case ends on: not "is my planner producing the optimal schedule?" but "what does my plant's average utilization actually look like, and how much of my weekly idleness is structural versus something I could sell into?"

That answer doesn't come out of a spreadsheet. It comes out of a model.

---

*Built with OptimEngine v9.0.0 on Google OR-Tools CP-SAT. The schedule and metrics presented here are the actual solver output for the input described, not retrospective approximations. The company name and customer specifics are synthetic; the operational pattern is drawn from years inside European mid-market contract manufacturing.*
