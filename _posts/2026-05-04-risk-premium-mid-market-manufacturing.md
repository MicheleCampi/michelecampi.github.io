---
title: "How fragile is your weekly plan? A risk-premium framework for mid-market manufacturers"
author: Michele Campi
published: true
description: "The optimal weekly plan from last article promised a makespan of 161 quarter-hours. But that number assumes durations are exactly what the planner says they are. What happens when reality varies by 20%? By 35%? Monte Carlo with CVaR turns the question 'is my plan robust?' into a quantitative one — and reveals that the right metric isn't fragility, it's the risk premium you pay for protection."
date: 2026-05-04
tags: [or-tools, scheduling, manufacturing, optimization, risk-management, monte-carlo, cvar]
---

**TL;DR.** A deterministic schedule promises a single number — say, 161 hours of plant time. That number assumes every task takes exactly as long as the planner wrote down. Real plants don't behave that way. Using Monte Carlo with CVaR 95% on a real OR-Tools schedule for a mid-market Italian contract packager, I show that doubling the input volatility (±20% → ±35% on filling tasks) raises the weekly *risk premium* from **4.2% to 7.2%** — not the catastrophic explosion most planners fear. The plan is structurally robust. The framework reproducible. The calculation is a single API call. *2,300 words, 8 minutes.*

---

In the last article I walked through one synthetic week at *Lombarda Confezionamenti SRL*, a fictional contract packager in northern Italy. OR-Tools returned an optimal weekly schedule with a makespan of 161 quarter-hours — about 40 hours of plant time, distributed across six lines and 26 production tasks. Roughly 15% better than the manual baseline, with zero late orders.

But there was a hidden assumption underneath that 161-hour number, and it's the assumption every deterministic schedule makes. It assumes that the duration of every task is exactly what the planner wrote in the spreadsheet. The crema viso filling lasts exactly 16 quarters. The shampoo run is exactly 28. The labelling on the body wash is exactly 14. No surprises. No variation.

In a real plant, of course, this is never true. Filling lines run a little faster on a good day and a little slower on a hard one. Validation sometimes takes an extra cycle when a new product comes in. Sanitization between formats is usually two hours but occasionally three. The week's actual makespan is some distribution around 161, not the number itself.

The question every operations manager intuitively asks is: **how robust is my plan?** Most of them answer it with stories — "last March we had a bad week, took us until Friday night," "the line 3 always runs late" — rather than numbers. Operations research can do better than that. It can quantify it.

## What "robust" actually means, mathematically

Two concepts from finance translate directly to scheduling under uncertainty: **Monte Carlo simulation** and **Conditional Value at Risk (CVaR)**.

The first is straightforward. Instead of assuming each task duration is a single number, we treat it as a probability distribution — say, triangular, with min/expected/max. We then ask the solver to evaluate the schedule against, say, 100 random samples drawn from those distributions. Each sample is a realistic "alternative week." The output is no longer a single makespan but a distribution of makespans: a histogram of how the week could actually play out.

The second concept matters more. **CVaR 95%** answers the question: *across the worst 5% of weeks, what is the average outcome?* Not the absolute worst case (which is dominated by tail events that may never happen), but the expected outcome conditional on being in the bad tail. If CVaR 95% of your weekly makespan is 168 hours when the expected value is 161, you can plan around 168 with reasonable confidence — not around 161.

Reframed in language a CFO understands: **the risk premium is the gap between the expected makespan and the CVaR 95%, expressed as a percentage of the expected.** It is the cost, in hours, of buying protection against bad weeks. A risk premium of 4% says: *to be safe in 95% of weeks, you need to budget 4% more time than the optimal plan suggests*. That number is concrete. It can be defended in front of a board. It can be priced into contracts.

## Scenario 1: a normal week (±20% volatility on filling tasks)

I ran the optimal schedule from the previous article through OptimEngine's stochastic scheduling endpoint. The setup: 100 Monte Carlo scenarios, triangular distribution on the four filling task durations (the structural bottleneck of cosmetics packaging), ±20% variation around the planned mean. Filling, not mixing or labelling, because filling is what happens on the shared bottleneck line and is most exposed to viscosity and product-changeover variability.

The output:

- **Mean makespan**: 161.38 quarter-hours
- **CVaR 95%**: 168.17 quarter-hours
- **Coefficient of variation**: 2.5%
- **Range** (min–max across 100 simulations): 152 to 169 quarters
- **Risk premium**: **4.2%**

What does this say? The plan is structurally robust. Even when the four filling tasks vary by ±20% — which is a generous estimate of normal-week volatility for a medium-complexity cosmetics line — the worst 5% of weeks land at 168 hours instead of 161. To buy 95% reliability you pay 4.2% extra time. That's the price of robustness.

Crucially, no week in the 100-simulation pool exploded. No catastrophic delay. No order missed. The schedule degrades gracefully — which is what we want, but rarely measure.

## Why is the plan this robust? The structural answer

Not all schedules degrade gracefully. Some shatter. The reason this one doesn't is structural, and worth explaining because it tells you when to expect different behavior.

The optimal plan from OR-Tools placed the four filling tasks across **two parallel filling lines**. When task A on line 1 runs 20% longer than expected, it doesn't ripple through tasks B, C, D on line 2 — they're on a separate machine. The week's makespan is determined by the *slower* of the two parallel paths, not the sum.

In other words, parallelism absorbs variance. A schedule that piles all four filling tasks on one line would have a much larger risk premium for the same input volatility — possibly 8-10% instead of 4.2%, because every variance compounds.

This is the kind of insight that's invisible without quantification. A planner looking at the schedule manually would say "looks fine." A solver looking at the schedule under stochastic perturbation says "looks fine, *and here's why*." That's what robustness analysis adds.

## Scenario 2: a difficult week (±35% volatility)

Now I doubled the volatility. ±35% on the four filling tasks — what you might see during a product-mix change, a new SKU introduction, or post-sanitization commissioning. Same schedule, same constraints, same 100 Monte Carlo simulations.

- **Mean makespan**: 161.28 quarter-hours
- **CVaR 95%**: 172.83 quarter-hours
- **Coefficient of variation**: 4.1%
- **Range** (min–max across 100 simulations): 144 to 173 quarters
- **Risk premium**: **7.2%**

Here is the surprising finding. **Volatility almost doubled, but the risk premium did not.** It went from 4.2% to 7.2% — a 71% relative increase, but in absolute terms still a manageable buffer. The mean makespan barely moved (161.28 vs 161.38). The structural robustness held.

This is not luck. It's the same parallelism story playing out. With two parallel lines, the worst-case outcome of the bottleneck path is bounded by the longer of two correlated random variables — whose expected maximum grows much more slowly than the underlying variance.

Compare this to a schedule that put all filling on one line: there, doubling input variance would roughly double the risk premium too, because the variances accumulate without offset. The plan would shatter.

## What this changes for the planner and the CFO

For the planner, this is a tool to **defend the optimal schedule against intuition**. If a senior operations manager says "I don't trust this plan, last March we had a terrible week" — the answer is no longer "trust me" or "it's optimal." The answer is "the framework projects a 4.2% risk premium under normal volatility, 7.2% under elevated volatility. Here are the numbers. Here is the assumption set. Here is what changes if you disagree with that assumption set."

That's a much harder conversation for the senior manager to win on intuition alone, because the framework is reproducible and falsifiable. If they disagree with the volatility input, they can change it. If they disagree with the parallelism modeling, they can override it. What they can't do is say "your plan is fragile" without engaging with the math.

For the CFO, the framing is different but equally concrete. The risk premium is a **cost**. It's the cost of buying delivery reliability. If the firm prices contracts on the deterministic plan (161 hours) and then absorbs the 4.2% slippage internally, that slippage shows up as overtime, expedited freight, or compressed margin on rush jobs. If the firm prices contracts on the CVaR plan (168 hours) and the week ends up at 161, that's bonus margin. Either way, the number is real and the conversation is honest. Without measurement, the firm pays the premium without knowing it exists.

## When this analysis does NOT apply

Three caveats matter for honest framing.

**First**, this analysis assumed parallelism in the bottleneck. If your plant has a single filling line, the framework still works but the risk premium will be larger and grow faster with volatility. The structural protection comes from the schedule's topology, not from the framework itself.

**Second**, only filling task durations were perturbed. In a real plant other parameters vary too — yields, setup times, downtime events. A complete robustness analysis would add stochastic distributions for those too. The ones we used are the dominant ones for cosmetics filling, but in another industry (precision machining, food processing, semiconductor assembly) the dominant variance source might be elsewhere.

**Third**, the analysis is conditional on the schedule produced by OR-Tools being itself near-optimal. If the input scheduling is poor, no risk analysis on top of it will save it. Robustness analysis is a layer over solid optimization, not a substitute for it.

## The framework, in five steps

This is the operational synthesis. Any mid-market manufacturer with weekly scheduling decisions can apply this:

1. **Compute the deterministic optimum** with OR-Tools or equivalent, using mean durations.
2. **Identify the dominant variance sources** — usually 2-4 task types where empirical historical variation is largest.
3. **Define triangular distributions** around each (min, expected, max) based on either historical data or expert estimate.
4. **Run 100 Monte Carlo simulations** with the same scheduler, and extract the makespan distribution. Compute CVaR 95% and the risk premium.
5. **Defend the schedule with the risk premium**, not the deterministic number. Price contracts, allocate buffer time, and have a real conversation with stakeholders.

What stays constant across industries, plant sizes, and product categories is the framework — and the fact that without measuring, the conversation about plan robustness is the same one production managers have been having for fifty years: based on intuition, on bad weeks they remember, on stories.

Operations research doesn't replace that intuition. It quantifies it.

---

*The analysis used OptimEngine's stochastic scheduling endpoint with 100 Monte Carlo scenarios. The two volatility profiles (±20% and ±35% on filling tasks, triangular distribution) were chosen to represent a normal week and a difficult week respectively. The full schedule, parameter distributions, and the resulting risk metrics are reproducible — the input was a single JSON request to a public endpoint, the output is what the solver returned.*

*If you're applying this framework to your own operation and want a second pair of eyes on the setup or the assumptions, the contact is on the profile.*
