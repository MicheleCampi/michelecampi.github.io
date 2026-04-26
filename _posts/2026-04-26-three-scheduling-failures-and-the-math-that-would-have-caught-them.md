---
title: "Three Production Scheduling Failures I've Seen, and the Math That Would Have Caught Them"
author: Michele Campi
published: true
description: Three chronic scheduling failures I watched repeat across cosmetic contract manufacturers in seven years of operations controlling. Each had the same shape: a visible symptom in the dashboards, a hidden cause nobody was tracking, a management reaction that addressed the symptom, and an operations research method that would have caught the cause. Notes for plant managers and operations directors who suspect their planning is leaving money on the floor.
date: 2026-04-26
tags: [scheduling, or-tools, manufacturing, optimization]
---

In seven years of operations controlling inside cosmetic contract manufacturers, I watched the same scheduling failures repeat across plants, across product lines, across teams. Different people, different machines, identical patterns.

The interesting thing about chronic scheduling failures isn't that they happen. It's that everyone in the plant knows they happen, and almost nobody knows why. Management explains them away with operational folklore — *"that machine is always troubled," "Friday afternoons are bad," "this client is impossible."* Engineers blame planners. Planners blame the ERP. The ERP blames the master data. Nothing changes.

What changed for me, slowly, was realizing that almost every chronic scheduling failure I'd seen had the same shape. There was a visible symptom that everyone agreed on. There was a hidden cause that nobody was tracking. There was a management reaction that addressed the symptom and not the cause. And there was, quietly waiting in textbooks nobody read, a piece of operations research math that would have solved it.

Three of those failures, anonymized but real, with the math that would have caught them.

---

## Failure 1 — The OEE that wouldn't move

Every contract manufacturer I worked with had at least one line where OEE was stubbornly stuck somewhere between 60% and 70%, far below the 85% management kept asking for. Production stoppages were frequent. Deadlines slipped every month, sometimes by hours, sometimes by days.

The dashboard showed the symptoms in red. The reaction was always the same: overtime shifts to recover, occasionally a maintenance contractor brought in to "fix the machine."

Neither solved anything, because neither addressed what was actually happening.

**The hidden cause was double.** First, micro-stoppages under five minutes weren't being tracked at all. The MES rounded everything below that threshold to zero. When we eventually instrumented the line and counted them properly, those untracked micro-stops were *roughly half* of the total downtime. The line wasn't failing dramatically. It was failing constantly, in tiny invisible bursts, and the cumulative effect was catastrophic.

Second, the preventive maintenance plan existed only on paper. Maintenance was scheduled every 200 hours in the manual, but in practice intervention happened reactively, after a failure had already caused a production stop. The plan was theoretical; the operation was firefighting.

**What the math would have done.** This is exactly the situation stochastic optimization was built for. Instead of treating the line's OEE as a deterministic input — *"machine X runs at 75% efficiency"* — you model micro-stops as a stochastic variable with the empirical distribution you observe. The Monte Carlo simulation generates thousands of possible production days, accounting for the real shape of the failure distribution, not its average.

The output isn't a number. It's a Conditional Value at Risk: *given the way this line actually behaves, what's the worst 5% of production days going to look like?* That single metric — CVaR 95% — would have ended the conversation about averages. Management would have seen, in numbers, that the line was vulnerable to compounding micro-stops in a way no average could capture.

For maintenance, the same engine answers a different question: when should preventive intervention happen to maximize expected uptime, given the empirical distribution of failure intervals? Not "every 200 hours" because that's what the manual says. The optimal interval, computed from data, often turns out to be very different.

The total infrastructure required to do this exists in OR-Tools and a Monte Carlo wrapper. It costs nothing in licenses. What it costs is recognizing that the problem isn't the machine — it's the model of the machine that everyone is using to make decisions.

---

## Failure 2 — The format change that took twice as long

Theoretical changeover time: two hours. Actual changeover time: four hours. Every single time, on every line, for years.

This wasn't a technical mystery. Everyone knew the SMED methodology existed. Some of the plants had even sent operators to training courses on it. The problem was that the management response to the gap between theoretical and actual time was always the same kind of response: structural and physical.

I saw companies dedicate specific lines to specific product families, sacrificing flexibility, to avoid the cross-format change problem entirely. I saw machine vendors brought back in to redesign accessibility on equipment that was perfectly accessible already. I saw line layouts modified, costing hundreds of thousands of euros, to shorten transport distances during changeovers.

What I never saw was anyone treating the changeover as what it actually is: **a scheduling and parallelization problem disguised as a hardware problem.**

**The hidden cause** was the framing itself. Management saw long changeovers as a sign that the machine was the bottleneck, when actually the bottleneck was the sequence in which changeover activities were performed and the fact that they were almost always serialized when they didn't need to be. Sanitization and tool retrieval and quality validation and operator briefing — these can all happen in parallel, with the right planning. Almost none of them did, in practice.

**The math that would have caught it.** Changeover sequencing is a classical Constraint Programming problem. CP-SAT solves it in milliseconds for problems up to a few hundred activities. You define each sub-activity (sanitize, retrieve tools, validate quality, brief operator, swap die, calibrate, run-in), the resource each consumes (operator A, operator B, the machine itself, the QC technician), and the precedence constraints (you can't run-in before calibration). The solver finds the minimum total time given these constraints, automatically discovering which activities can run in parallel and which must wait.

For a typical contract manufacturer running roughly five format changes per month per machine, with two hours of avoidable extra time per change and an industrial cost of line idleness in the €80–€120 per hour range, the financial impact of leaving this unsolved tends to land somewhere between €40K and €60K per year per plant. Not catastrophic. Not invisible either. The kind of number that, once quantified, makes the conversation about "should we invest in optimization" finally productive.

The other piece of math — sequence-dependent setup times — addresses what comes before. Two consecutive products with similar fragrances need a brief sanitization. Two products with very different formulas need a deep cleaning that takes triple the time. Treating all sanitizations as uniform in the schedule is one of the most expensive simplifications I've seen in cosmetic manufacturing. CP-SAT handles asymmetric setup times natively. Spreadsheets don't.

---

## Failure 3 — The customer who could move anything

The third pattern is the one that's most specific to contract manufacturing, and I think the least discussed publicly. It has to do with the fact that contract manufacturers don't really plan their own production. Their customers do.

In every plant I worked with, somewhere around 15% of orders were modified in the seven days before production. Quantities changed. Formulas were tweaked. Delivery dates moved up because of a marketing campaign nobody had mentioned earlier. Sometimes the customer would call on Thursday asking to bring forward a batch scheduled for the following Monday, and the answer was always yes, because the customer was big and the relationship mattered.

**The hidden cause was triple, and worth unpacking carefully.**

Large customers knew they could move deadlines and they did so systematically. There was an asymmetry of leverage that nobody had quantified, so nobody had pushed back on. Smaller customers anticipated batches for marketing reasons without understanding the impact on line saturation — they weren't being malicious, they simply lacked the information that would have made them think twice. And internally, *nobody calculated the real cost of a last-minute change*. The commercial team accepted modifications because to them, it was free. The cost showed up later, in overtime hours and missed deadlines on other clients' orders, and by then it was diffused enough that you couldn't trace it back to the original "yes."

**The management response** was almost always to schedule a planning-commercial meeting where rules of engagement were established for accepting last-minute changes. Those rules then proceeded to be ignored within a month. I don't say this with cynicism — the rules were genuinely well-designed, in some cases. They just couldn't survive contact with the reality that the commercial team's incentive structure rewarded saying yes and the planning team had no quantitative argument for saying no.

**The math is two pieces.** First, robust optimization under demand uncertainty. Instead of producing a single deterministic schedule and reacting to changes one by one, you produce a schedule that's robust against the typical pattern of last-minute modifications. The solver knows that 15% of orders will change, and it finds the schedule that minimizes worst-case impact across plausible modification scenarios. The schedule itself becomes more resilient — built-in slack on the right resources, batch ordering that survives reordering, buffer placement informed by which customers historically modify and which don't.

Second, and this matters more than the first: **automated cost-of-change quantification**. When the customer calls on Thursday asking to bring a Monday batch forward, the planner shouldn't have to estimate the impact in their head. The system should respond with a number: *"This change costs us roughly €X in disrupted capacity and pushes three other deliveries by Y hours. Are we accepting this, and what's the customer paying for it?"* The cost stops being invisible. The conversation shifts.

This is exactly what sensitivity analysis on top of a scheduling solver does. Perturb one parameter (the deadline of order #347), re-solve, observe the delta in the objective function. The technology to do this in real time is mature. What's missing is almost never the technology. It's the recognition that the planning team needs to bring numbers to the meeting, not opinions.

---

## What ties these three failures together

I want to be careful not to oversimplify, because each of these problems is real and complicated and deserves more than a paragraph of treatment. But there's a pattern worth naming.

In all three cases, the planning team was operating with a model of reality that was missing something. Micro-stops were missing from the OEE model. Parallelizable activities were missing from the changeover model. Customer modification probability was missing from the demand model. Each missing piece led to decisions that looked reasonable on the dashboard and were quietly destructive in production.

Operations research math doesn't fix these failures by being magical. It fixes them by forcing you to write down the model — explicitly, in code — and then revealing the parts of reality the model is ignoring. The discipline isn't the optimization. It's the modeling.

The tools to do this kind of modeling are mature, free, and well-documented. OR-Tools handles all three of the math problems I described. Python or any language with a CP-SAT binding is sufficient. The hard part isn't the technology.

The hard part is having someone in the organization who has spent enough time on the production floor to know which parts of reality the spreadsheet is hiding, and enough time with the math to know which solver to point at the problem.

That intersection is rare. It's also where most of the value is.

---

*I build production optimization tooling and consult with European mid-market manufacturers who want to move past spreadsheet-based planning. If any of these patterns sound familiar, the contact links are in the site header.*
