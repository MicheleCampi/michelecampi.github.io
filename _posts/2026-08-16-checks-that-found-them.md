---
layout: post
title: "Six things that were wrong, and the checks that found them before anyone else did"
subtitle: "A GPU campaign, a translation, a config for someone else's codebase. None of the defects were interesting on their own — what is interesting is that in every case the thing that caught them was cheap, and the thing that would have caught them late was not."
date: 2026-08-16
categories: observability systems-engineering llm-inference
tags: rust llm profiling observability methodology experiments testing
---

The last post reported a curve: on an agentic trajectory the cost of generating holds flat while the cost of waiting on tools grows 25×, and the obvious policy for reclaiming that idle time repays on none of fifteen cells.

This one is about how that number survived contact with me.

Six defects turned up while producing it — in the measurement code, in the analysis, in the write-up, and later in a config written against someone else's codebase. Individually they are unremarkable: an arithmetic slip, a wrong constant, a rounding. What makes them worth a post is the pattern in how they surfaced. Every one was caught by something that cost minutes. Every one, uncaught, would have cost either a GPU session, a published figure, or credibility with a maintainer.

None of this is novel practice. It is the ordinary discipline of measurement, applied to a small solo project where nothing forces it — no reviewer, no incident, no customer. That is precisely why it is worth writing down: on a two-person team the process catches these; alone, only a habit does.

## The rule underneath all six

State the thing that would make you wrong, then go look.

Not "test your code" — that catches the defects you anticipated. The useful version is narrower: for each figure or decision, name what would have to be true for it to be wrong, then build the cheapest thing that can tell you. Often that is not a test. Twice below it was a script that compares two files. Once it was multiplying four numbers together.

The corollary is the part people skip: **a check that has never failed is not yet a check.** Two of the six below were found only because I deliberately broke something to see whether the check noticed. One of them didn't.

## 1. A number that was right until a flag made it wrong

The harness recorded how long each trajectory spent waiting on tools as `n_tool × tool_latency_s` — the number of tool calls times the requested latency. Correct, for as long as every sleep lasted exactly the mean.

Then the sweep gained a coefficient-of-variation flag, so the sleeps were drawn from a lognormal instead of fixed. The product kept computing. Three draws averaging 0.4s summed to 1.5s and were recorded as 1.2s.

What makes this the dangerous kind: 1.2 against 1.5 does not look broken. It is the right order of magnitude, it moves in the right direction, and it appears in a field named `tool_wall_s` that nobody re-derives. And it was the numerator of the non-generating fraction in exactly the cells spent measuring dispersion — the ones where the figure mattered most.

Found by writing a test for a module that had none. The module was `trajectory_gates.py`, which decides whether a paid GPU cell's evidence is usable at all: the piece of code with the highest consequence per line in the whole harness, and the only one with zero coverage. That is not a coincidence — it had no tests *because* it felt obviously correct.

The first version of the test passed against the broken code. I had picked draws averaging exactly to the parameter, so sum and product agreed. The test only became a test when the draws were chosen so the sample mean deliberately differs from the requested one — which is what real sampling does with n=3.

## 2. A `0` that was the only value that could hide the bug

The cell orchestrator prints every argv before spending anything, so what the dry-run shows is what will run. That is the whole point of building it in Python rather than shell: both come from the same list object, so they cannot drift.

They drifted anyway, in one place. The sampling window was parsed as a float, and `argparse` renders 45 as `"45.0"`. The profiler's `--duration-secs` rejects a decimal point.

The dry-run had been exercised many times and never caught it, because during dry-run the window is unset and prints as `0` — the one float that formats as an integer. The defect was invisible precisely where it should have been visible.

Found by running the whole cell against a local llama.cpp before booking a GPU. Not a rehearsal of the logic: a rehearsal of the *invocation*, with the real binary rejecting the real string.

## 3. Two figures that were wrong in opposite directions

Both found by regenerating every number in a finished README from the archived evidence, instead of copying them across from where they were computed.

A table labelled "mean per latency" carried the seed-42 replica: $0.009111 where the mean was $0.009127. Small, and wrong in a way no reader could detect.

The other was worse because it was deliberate. Growth across the sweep is +55.5%, and I wrote +55%, rounding down on the grounds that understating your own result is the cautious direction. It is not. Rounding against your own measurement is not caution, it is a wrong number that happens to flatter you less — and a reader who recomputes finds a discrepancy either way. It became +56%.

The same pass produced a third change that was not a defect but an improvement of the same kind: "the cost of generating is flat at $0.00915" became "$0.00911–0.00916, a 0.5% band". On a claim that a quantity does not move, *how much* it does not move is the claim. A rounded single figure asserts more precision than the measurement has.

## 4. A translation that could have moved a decimal point

The protocol document is 560 lines carrying 333 numbers, and it had to move from Italian to English. Rereading 560 lines against 560 lines is the obvious approach and a bad one: the eye reads what it expects, and the figures are six-decimal costs and thousands separators that invert between the two languages — `$0.009127` against `$0,009127`, `251,320` against `251.320`.

So before translating anything, a script that extracts the multiset of numbers and identifiers from both files and reports what appears in one and not the other. Multiset, not set: a figure that survives but is repeated the wrong number of times is exactly what a reread misses.

It earned itself three times. Twice it flagged a figure present in the English with no counterpart in the Italian — which reads like a fabricated number until you look, and both times was my extraction range stopping short of a table's last rows. Once it was a real difference: "1 parola hex" had become "one hex word". Correct English, and a lost digit in a document where every quantity is a measurement.

Building the extractor also produced a small lesson in the same shape as the rest. Its first version had a lookbehind meant to keep `A10` and `ADR-011` out of the number stream, and that lookbehind also rejected a digit following a hyphen — so the range `1,20-1,26` yielded `1,20` and a bare `26`. It broke the case it was added for, and I found that by running it against the document rather than reasoning about it.

## 5. An experiment that was undecidable before it was run

An ADR designed an experiment to test whether a measured bound holds as a capacity figure: drive N trajectories concurrently, observe the engine's running-request count, and compare it against `N × (1 − f_nongen)`. The falsification criterion was fixed in advance — within 10% of N rules the bound out, within 15% of the prediction supports it.

Both halves were reasonable. Together, at three of the four latencies the campaign had measured, they were undecidable: the bands overlap, so a single observed mean satisfies both criteria at once and the experiment returns two opposite verdicts on the same number.

That is not noise. Replicas do not fix it, longer runs do not fix it, and it would have been discovered on a rented GPU with the arms already spent.

Found by multiplying four numbers before booking anything. The gap between the two thresholds is `0.9N − 1.15N(1 − f_nongen)`, which is negative unless `f_nongen` is large enough — 2.30%, 5.55% and 19.01% all fail, 37.01% clears by 0.351.

The fix was to move the cell, not the thresholds. Widening the 15% to make a lower-latency cell decidable would be choosing the criterion to fit the data, which is the thing declaring criteria in advance exists to prevent.

## 6. A check that could not fail, and the one that could

The last is the one I would tell someone starting out.

Working on a config for a project I do not own, I needed to know whether it was correct before proposing it. The project has a two-phase loader — one phase decodes and validates references, the other builds the produce/consume graph between plugins — so I wrote a test that loads the config through both. It passed.

Then I deleted the block the config exists to provide, expecting a failure. It passed again.

The reason is in the plugin's own contract: the scorer declares its data dependency as `Optional`, and the framework's scoping only builds the set of keys a plugin is permitted to touch. Nothing verifies that a producer exists for what a consumer reads. So a config missing the piece that publishes the data loads cleanly, runs, and scores every endpoint zero — indistinguishable from a scorer that discriminates badly, and a benchmark would report it as a result.

The test went in the bin. What replaced it compares the two implementations behaviourally: same endpoints, values carried in both the representations the two paths read, score maps must agree. Delete one of the two and it fails with `legacy=1` against `ea=0` — which turned out to be a semantic difference worth reporting on its own, since a missing value reads as *best* under one implementation and *worst* under the other.

Two lessons in one episode. A check that has never failed is not yet a check. And when a check refuses to fail, the interesting question is not how to fix the test — it is what the system is telling you about itself.

## What the six have in common

**The check is almost always cheaper than the thing it protects.** A test on a module with zero coverage, a rehearsal against a local engine, a script comparing two files, four multiplications. Against: a wasted GPU session, a published figure that does not reproduce, a config a maintainer would have had to catch for me.

**The dangerous defects are the plausible ones.** Not the crash — the crash announces itself. 1.2 where 1.5 was right, `45.0` where `45` was needed, a mean labelled as a mean and holding one replica. All of them look fine, none of them fail loudly, and all of them survive a careful reread.

**Three of the six were in the checking, not in the work.** A test that passed against broken code. A dry-run blind at exactly the one value that mattered. A loader test that could not fail. If the habit is "add a check and move on", these stay invisible — and the false confidence is worse than no check, because it stops you looking.

**Two were found only by deliberately breaking something.** That is the cheapest technique in this list and the one most easily skipped: after writing a check, break the thing it guards and confirm it notices. It takes a minute and it is the only way to know whether you built a check or a decoration.

## What this costs

Perhaps a day across a two-week campaign. The GPU session it protected cost $0.55 and twenty-five minutes; the one it *did not* protect, three days earlier, was abandoned after $0.90 on four environmental problems that a pre-flight would have caught — which is where the checklist entries came from.

That asymmetry is the argument. Not rigour as virtue: rigour as the cheaper option, once you count the runs you do not have to repeat and the numbers you do not have to retract.

The campaign these came from is [in the previous post](/observability/systems-engineering/llm-inference/2026/08/06/agentic-trajectory-cost.html); the harness, the evidence and the ADRs are public in [agentic-kv-energy-experiment](https://github.com/MicheleCampi/agentic-kv-energy-experiment) and [vllm-coldstart-operator](https://github.com/MicheleCampi/vllm-coldstart-operator). Every defect above is in the commit history under its own message, which is the only form of this post that will still be accurate in a year.
