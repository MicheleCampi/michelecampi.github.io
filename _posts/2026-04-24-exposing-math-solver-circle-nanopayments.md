---
title: "Exposing a math solver as Circle Nanopayments: what I learned forking arc-nanopayments"
date: 2026-04-24
tags: [x402, circle, nanopayments, agent-economy, nextjs, or-tools]
---

# Exposing a math solver as Circle Nanopayments: what I learned forking arc-nanopayments

I spent the last three weeks wiring a constraint-programming solver into Circle's Nanopayments stack on Arc testnet. The result is [`optim-arc-v3`](https://github.com/MicheleCampi/optim-arc-v3), a Next.js gateway that exposes ten optimization endpoints — scheduling, routing, Pareto frontiers, stochastic CVaR analysis — each behind a `402 Payment Required` response that accepts gasless USDC micropayments.

It's live at [optim-arc-v3.vercel.app](https://optim-arc-v3.vercel.app). You can hit any endpoint with `curl` right now and get back a valid x402 v2 payment challenge.

This post is about how I got there. Not the Circle marketing pitch — the actual friction, the choices I didn't expect to face, and the patterns I'd reach for next time.

## Why this, why now

In March 2026 Circle published a blog post on Nanopayments. I was already tracking them for stablecoin infrastructure reasons, but that post clicked something. Sub-cent, gasless USDC payments batched off-chain and settled on-chain periodically — this wasn't another "blockchain primitive in search of a use case." It was plumbing for a thing I'd been thinking about for months: **markets of calculations and decisions**.

Here's what I mean. I run OptimEngine, a mathematical optimization service built on Google OR-Tools. It solves things like factory scheduling, logistics routing, resource packing — problems that LLMs cannot solve, because they require constraint satisfaction and provable optimality, not pattern matching. My hunch is that agentic systems in 2026-2027 will increasingly need to call into specialized solvers for decisions that matter. Not every problem is a text generation problem.

But for an AI agent to *pay* my solver for each call — autonomously, without human approval loops, at sub-cent granularity — the payment layer has to disappear. Traditional APIs with credit card billing and monthly invoices don't work when the caller is a machine making ten thousand decisions per hour. That's the gap Nanopayments targets.

So the plan crystallized: make OptimEngine speak x402 natively on Arc, Circle's payment-native chain. An agent sees a `402`, signs an EIP-3009 authorization off-chain, retries, gets the solver response. Zero gas. Settlement batched by Circle in the background. For the agent, it's just an HTTP call with a payment header.

## The decision: fork, don't build from scratch

I already had a working gateway on Arc testnet — an Express app I'd hand-rolled months earlier, implementing a custom x402 flow against the same network. One transaction had even been processed through it. Extending that code was the obvious path.

I chose the opposite: fork Circle's own [`arc-nanopayments`](https://github.com/circlefin/arc-nanopayments) sample and adapt it.

The reasoning, which I worked through in iterative discussion with an AI assistant (a workflow I've come to rely on for significant architectural choices): Circle's sample is the reference implementation. It's the code their DevRel team points to, built on their own SDKs, validated against their own infrastructure. Starting from something they promote means inheriting their assumptions about batching, signature verification, and settlement — assumptions I'd otherwise have to reverse-engineer.

The trade-off was real. Their sample is Next.js + Supabase + Tailwind, while my existing gateway was Express + ethers. Different stack. Different deployment story. Adopting it meant throwing away a working codebase.

I estimated the fork would be "90% compatible" with what I already had. That was optimistic. When I actually walked through the code, it was closer to 50-60%. The payment flow patterns matched at a high level, but every implementation detail — how the `payment-required` header is structured, how signatures are verified, how settlement is recorded — was different enough to require new code.

What saved the approach wasn't the estimate being accurate. It was the ability to course-correct quickly. The principle that emerged, which I'll keep reusing: **what matters is moving quickly from misalignment to alignment**. An imprecise initial estimate isn't a failure mode if the process catches it early and adjusts.

## The implementation: thin proxy, HOC pattern

The core insight, once I'd read Circle's sample carefully, is that the x402/Nanopayments flow is encapsulated in a single Higher-Order Function: `withGateway`. You write a normal Next.js route handler that returns JSON, then wrap it:

```typescript
import { NextRequest, NextResponse } from "next/server";
import { withGateway } from "@/lib/x402";

const CORE_GATEWAY_URL = process.env.CORE_GATEWAY_URL;
const CORE_GATEWAY_KEY = process.env.CORE_GATEWAY_KEY;

const handler = async (req: NextRequest) => {
  const body = await req.json();

  const coreRes = await fetch(`${CORE_GATEWAY_URL}/pack-resources`, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "X-Core-Key": CORE_GATEWAY_KEY,
    },
    body: JSON.stringify(body),
  });

  const responseBody = await coreRes.text();
  return new NextResponse(responseBody, {
    status: coreRes.status,
    headers: { "Content-Type": "application/json" },
  });
};

export const POST = withGateway(handler, "$0.25", "/api/solve/pack-resources");
```

That's it. The handler is a thin proxy to my existing orchestration layer (the OptimEngine Core Gateway, running separately). It doesn't know anything about x402, Nanopayments, Arc, or USDC. Payment verification, settlement, and event logging to Supabase all happen inside `withGateway`.

When a client hits this route without a payment header, the response is `HTTP 402 Payment Required` with a base64-encoded `payment-required` header. Decoded, the JSON looks like this (shortened):

```json
{
  "x402Version": 2,
  "accepts": [{
    "scheme": "exact",
    "network": "eip155:5042002",
    "asset": "0x3600000000000000000000000000000000000000",
    "amount": "250000",
    "payTo": "0x389D73e1cAC5e4D4a7BB3C6c4Cf35aB36bF00712",
    "extra": {
      "name": "GatewayWalletBatched",
      "verifyingContract": "0x0077777d7EBA4688BDeF3E311b846F25870A19B9"
    }
  }]
}
```

The `network` field identifies Arc testnet in CAIP-2 format. The `amount` is in USDC atomic units (6 decimals, so 250000 = $0.25). The `extra.name: "GatewayWalletBatched"` is the flag that tells a compatible client to use Circle's batched settlement path via the Gateway Wallet contract at `0x0077777d7EBA...` — this is what makes the payment gasless and nanopayment-scale.

A client that understands this responds by signing an EIP-3009 authorization off-chain, base64-encoding it into a `payment-signature` header, and retrying. The seller's `withGateway` wrapper forwards the signature to Circle's `BatchFacilitatorClient.verify()` and `BatchFacilitatorClient.settle()`, records the payment event in Supabase, then calls my handler and streams the response back.

I did this pattern once for `/api/solve/pack-resources`, then generated the remaining nine endpoints with a shell script that templates the same ~30-line proxy. Each endpoint has a different price tier and proxies to a different route on my Core Gateway, but the middleware logic is identical.

## The fork, minimally

The upstream sample includes a dashboard UI, a LangChain-powered agent buyer, and four demo endpoints that return things like motivational quotes and toy JSON datasets. I kept none of it.

Removed:

- `agent.mts` and the LangChain/DeepAgents/OpenAI dependencies. I'm building a seller, not a buyer. The sample's buyer agent is instructive but it's infrastructure I don't need to own.
- `app/dashboard/` and the related React components. If I ever want a seller dashboard, I'll build one tailored to OptimEngine metrics. The sample's dashboard was for demonstrating Circle's balance and withdrawal APIs, which aren't the story I want to tell.
- The four demo paywalled routes.

Kept:

- `lib/x402.ts` — the core middleware. Circle's implementation is well-structured, uses the official SDK, and handles edge cases I'd otherwise forget. Apache-2.0 licensed.
- `supabase/migrations/` — the schema for `payment_events` and `withdrawals` tables.
- The `app/api/gateway/balance` and `app/api/gateway/withdraw` endpoints. I don't use them today, but leaving them costs nothing and saves work later.

The result was a clean install of 189 packages (down from 327 in the upstream sample), zero vulnerabilities, TypeScript passing with no errors. First commit on April 23, production deploy on Vercel two days later.

## Deploy: Vercel, and one thing I'd redo

Vercel was the right choice for me, even though my other infrastructure is on Railway. Next.js 16 with Turbopack is Vercel-native — zero configuration, push to `main`, live in ninety seconds. The free tier's 10-second function timeout is a real limit on my heaviest endpoint (a composite analysis that can take seven seconds of solver time), but it's a constraint I'll manage rather than architect around for now.

The one thing I'd do differently: iterate on the Vercel setup more carefully. I accidentally created two parallel Vercel projects pointing to the same GitHub repo — one where I'd configured environment variables correctly, one left empty. Builds on the empty project kept failing with `supabaseUrl is required` errors that looked identical to missing configuration. It took a screenshot comparison to realize I was looking at two different projects in my dashboard. The lesson wasn't technical. It was: when an error repeats after you think you've fixed it, check that you're fixing the right instance.

Once sorted, the smoke test from my terminal was clean: all ten endpoints returning `HTTP 402` with valid payload, response times between 370 and 730 milliseconds, first call warm-up aside.

## Two things I'd want you to take from this

**First, on the practical side: don't be afraid to fork enterprise code.** Circle's sample isn't sacred. It's Apache-2.0 licensed, it's designed to be adapted, and the maintainers at Circle would rather see external builders fork it into real applications than admire it untouched. Strip what you don't need, keep what you do, credit the upstream in your README, and move on. The sample is a starting point, not a template to preserve.

**Second, on the broader bet: x402 + Nanopayments is foundational infrastructure for agent economies.** Not because it's fashionable, but because the alternative — traditional payment rails, human-approved transactions, monthly billing cycles — doesn't scale to the kind of economic activity that autonomous agents will generate. If an agent wants to call my solver ten thousand times to evaluate a decision space, the payment layer has to be invisible and nearly free. That's Nanopayments. On Arc, with USDC, through Circle's Gateway, it works.

The piece that convinced me this was worth three weeks of effort wasn't the technology itself. It was the realization that **markets of calculations and decisions** — where agents pay specialized services for mathematical work at micropayment scale — become economically viable once the payment friction collapses. Solvers, predictors, validators, oracles: all of them become addressable by autonomous clients who can pay per call without accounting overhead. The solver is just the first primitive I happened to have ready.

## Try it, or reach out

The endpoint is live. A single command:

```bash
curl -i -X POST https://optim-arc-v3.vercel.app/api/solve/pack-resources \
  -H "Content-Type: application/json" \
  -d '{}'
```

You'll get back a `402` with a decodable `payment-required` header. Everything to implement a client is in Circle's [Nanopayments documentation](https://developers.circle.com/gateway/nanopayments).

If you're building an agent that needs to make optimization decisions — scheduling, routing, portfolio construction, resource allocation — and you want to integrate OptimEngine as a solver in your pipeline, reach out. The solver is production-grade, the payment rail is now Circle-native, and I'm interested in hearing what kinds of problems you're trying to make agent-solvable.

---

*Built on top of [circlefin/arc-nanopayments](https://github.com/circlefin/arc-nanopayments) (Apache-2.0). Thanks to the Circle team for the sample and to Arc network for making sub-cent USDC payments practical on testnet. The OptimEngine Core Gateway is private infrastructure that orchestrates Google OR-Tools CP-SAT solvers behind this payment layer.*
