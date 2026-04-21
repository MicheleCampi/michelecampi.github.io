---
title: "How I exposed OR-Tools as a production MCP server"
author: Michele Campi
published: true
description: A solo builder's account of wrapping Google's OR-Tools CP-SAT solver as a dual-transport MCP server with OAuth 2.1, rate limiting, and agent-native discovery. Includes the debugging saga that got it working.
tags: [mcp, ortools, oauth, fastapi]
---

# How I exposed OR-Tools as a production MCP server

In April 2026, the tooling gap is starting to show.

AI agents can generate text, call APIs, navigate browsers, write and execute code. They can do a lot. What they can't do reliably is decide how to schedule ten manufacturing jobs across three machines so that lateness is minimized. Ask a language model to do it and you'll watch it reason for forty-five seconds, then return a plausible-looking sequence that's noticeably worse than optimal. Or worse, confidently wrong.

OR-Tools CP-SAT, Google's open-source constraint solver, returns the provably optimal schedule for the same problem in under fifty milliseconds. No hallucination. No "let me reason about this step by step." Just a deterministic answer with a mathematical guarantee.

So I spent the last two months wrapping it as a production MCP server that autonomous agents can call. This is how it went, what I chose, what I'd do differently, and the bugs I hit — all documented in a git log that I later had to clean up.

## The problem I came from

I'm not a "tech guy" by origin. I've spent seven years as an operations controller in cosmetic contract manufacturing in northern Italy — the kind of work where you watch production managers rebuild weekly schedules by hand every Monday morning because the MES system doesn't do real scheduling, it only tracks events after the fact.

The scheduling problems I saw were always the same shape. Five to fifteen machines. Ten to twenty active jobs with different customer deadlines and setup dependencies. A plant manager with an Excel sheet and twenty years of gut feeling. When something went wrong — a line didn't start, a machine broke down — the replan was done by eye, because formally recalculating would take hours. The cost of a late order translates into penalties, lost customer goodwill, and sometimes lost customers.

Commercial alternatives existed but weren't accessible. Enterprise MES and APS vendors start in the low six figures per year in license fees, often multiples more once you include integration and consulting. For a small-to-medium manufacturer in Emilia or Lombardy doing ten to thirty million in revenue, those numbers don't pencil out.

Before OptimEngine, I built smaller things: an order forecast scheduler for my controller work (Gantt visualization, multi-job parallelism, economics tracking), then a routing demo. Neither went anywhere, but they taught me the pattern: **the math that enterprise software sells for six figures runs on open-source solvers that anyone can use, if they know how**.

Then MCP landed. Anthropic released the spec, and I started noticing that agents building on top of frontier LLMs could now *discover* tools via a standardized protocol, not hard-coded integrations. The question answered itself: what if I took a world-class solver and made it an MCP tool that any autonomous agent could call without knowing anything about operations research?

That's what OptimEngine is now.

## Why MCP, not just a REST API

The first architectural decision was whether to build this as a conventional REST service or as an MCP server. I ended up doing both. Here's the reasoning.

REST APIs are universal. They work from anywhere. But they have one structural limitation in the agent era: **the agent has to know you exist before it can call you**. Every REST integration requires someone — a developer, a prompt engineer — to manually wire the API into the agent's context. You are a service that needs to be introduced.

MCP flips that. An MCP server exposes a tool manifest: a machine-readable description of what it does, what inputs it takes, and what outputs it returns. Agents can discover MCP servers through registries like Smithery, read the manifest, and decide on the fly whether to invoke the tool. No human integration step.

For a solver service, this matters. If the target user is "every AI agent that might ever need to solve a scheduling problem," the ratio of developers-who-will-write-custom-integrations to agents-that-will-find-me-via-MCP-and-call-me tips hard toward MCP over a three-year horizon. MCP is where the audience is going.

But REST still earns its place. My own use cases — a Next.js SaaS for Italian SMEs called PMI Scheduler, server-to-server workflows, Stripe-based billing — all speak REST natively. Forcing them through MCP would be silly.

The solution is dual-stack. Same FastAPI backend, same OR-Tools core. REST for traditional integrations, MCP for agent-native consumers. Routes split by transport, logic shared by the solver layer underneath.

## Architecture

Here's the shape of OptimEngine at a high level:

```
REST client             MCP client
(Next.js proxy,        (Claude Desktop,
 cURL, Python SDK)      Cursor, autonomous agent)
       |                    |
       v                    v
  FastAPI app          MCP transport
  (api/server.py)      (/mcp SSE +
                        /mcp/v2 Streamable HTTP)
       |                    |
       +---------+----------+
                 v
         Solver dispatch
         (solver/, routing/, packing/,
          pareto/, robust/, stochastic/)
                 |
                 v
         OR-Tools CP-SAT
                 |
                 v
            JSON response
```

The application is FastAPI. I went with it because it's what I already knew from previous projects — Python async, auto-generated OpenAPI docs, dependency injection that fits naturally with middleware. Flask would have worked; Starlette alone might have been lighter. I didn't benchmark the alternatives, and I don't regret it.

Under the FastAPI layer sits a modular solver tree. Eight domains: scheduling, routing, packing, pareto frontier, prescriptive analytics, robust optimization, sensitivity analysis, stochastic scheduling. That breadth emerged incrementally. I started with scheduling, iterated with AI pair-programming, kept adding domains as I understood the space better.

In retrospect, this is both the strength and the weakness of the project. Scheduling and routing are the most mature and most used. Pareto and sensitivity are production-grade. The robust and stochastic modules exist but haven't been stress-tested by real traffic. If I were starting again, I'd build depth in one domain first and expand only after the first one had paying users. Shipping eight domains in two months looks impressive but makes each one thinner than it could be.

The solver choice — CP-SAT specifically — was deliberate. OR-Tools offers multiple solvers (linear programming, mixed-integer, constraint programming), but CP-SAT is the sweet spot for the problems I care about: combinatorial scheduling with hard constraints, vehicle routing with capacity windows, bin packing with rules. It's open source, it's been battle-tested by Google internally, and it handles disjunctive constraints natively — something MIP solvers tend to struggle with. I didn't run head-to-head benchmarks against Gurobi or CPLEX. I chose CP-SAT on the recommendation that it fits this problem class, and it has.

## Dual MCP transport: SSE and Streamable HTTP

MCP supports multiple transports. The older one is Server-Sent Events (SSE): long-lived HTTP connections where the server streams tool invocations to the client. The newer one is Streamable HTTP, which moves toward a more conventional request/response model with first-class support for auth.

OptimEngine deploys both.

```python
# Open, rate-limited, no auth — for demos and compat
mcp.mount_sse(mount_path="/mcp")

# Streamable HTTP + OAuth 2.1 — for production agents
if _SCALEKIT_CONFIGURED:
    mcp.mount_http(mount_path="/mcp/v2")
```

The `/mcp` endpoint is open, rate-limited at ten tool calls per hour per IP via a custom middleware. Good enough for Claude Desktop, Cursor, and anyone kicking the tires on the service. No authentication, no billing, no friction.

The `/mcp/v2` endpoint requires a valid OAuth 2.1 bearer token, validated against a ScaleKit JWKS endpoint. This is the tier where production agents with real traffic live.

**A gotcha I spent time on**: the `mcp` Python library's `mount_sse()` method defaults to mounting at `/sse`, not `/mcp`. The older `mount()` method used `/mcp` as its default. If you're upgrading from an earlier version and don't pass `mount_path="/mcp"` explicitly, every existing client breaks silently — the SSE handshake succeeds at the wrong path, and nothing in the logs makes this obvious. The fix is one argument, but the debugging time to find it is real. I learned this the hard way during post-deploy verification.

**A second gotcha, related to auth**: the `/.well-known/oauth-protected-resource` endpoint — the standard OAuth 2.1 discovery path that MCP clients call before authentication — was being blocked by my API key middleware because it didn't match any of the public paths. Clients would get a 403 during discovery, fail to initialize OAuth, and give up. The fix is to add `/.well-known` to the middleware's bypass list. Trivial in hindsight, but it cost a deploy cycle to figure out.

If you're building your first MCP server: start with SSE, keep it open, add auth and the streamable transport only when you have a concrete reason. My mistake early on was adding OAuth before I had any user who needed it.

## Authentication without building an OAuth server

Implementing OAuth 2.1 from scratch is something nobody sensible does anymore. The RFCs are dense, the attack surface is large, and the table stakes for correctness are high. The pragmatic choice is to delegate to an identity provider.

I used ScaleKit. The tier is free for development, it supports OAuth 2.1 natively, it handles Dynamic Client Registration — which MCP discovery platforms like Smithery require — and the dashboard is usable. Auth0 would have worked too, Clerk probably, plenty of others.

The integration is lean: four environment variables, a JWT validator using PyJWT directly, and a FastAPI dependency that checks the bearer token on every `/mcp/v2` request.

```python
from jwt import PyJWKClient, decode as jwt_decode

_JWKS_CLIENT = PyJWKClient(
    f"{_SCALEKIT_ENV_URL}/.well-known/jwks.json",
    cache_keys=True,
    lifespan=3600,
)

async def validate_mcp_bearer(request: Request) -> dict:
    auth = request.headers.get("Authorization", "")
    if not auth.startswith("Bearer "):
        raise HTTPException(401, "Missing bearer token")
    token = auth[7:]
    signing_key = _JWKS_CLIENT.get_signing_key_from_jwt(token).key
    try:
        claims = jwt_decode(
            token,
            signing_key,
            algorithms=["RS256"],
            audience=_SCALEKIT_RESOURCE_ID,
        )
    except Exception as e:
        raise HTTPException(401, f"Invalid token: {e}")
    return claims
```

**A nontrivial gotcha here**: the official `scalekit-sdk-python` package depends on protobuf 5.x. OR-Tools, the entire reason this service exists, depends on protobuf 6.33+. The two cannot coexist in the same Python environment. I wasted a deploy figuring this out before switching to PyJWT directly against the JWKS endpoint. Public-key validation is the only thing I needed from the SDK anyway — no need for the full client library.

This is a recurring pattern in modern Python backends: SDKs from auth providers pull heavy dependency trees that conflict with whatever scientific or ML libraries you're also using. When it happens, drop the SDK and call the provider's raw HTTP endpoints. OAuth 2.1 with JWKS is simple enough to do in thirty lines of code.

## The Smithery issuer match saga

The last piece — making OptimEngine discoverable on Smithery, the de-facto MCP directory — turned out to be the hardest. It's worth describing because it illustrates how young this ecosystem still is.

Smithery tries to connect to an MCP server, complete the OAuth handshake, and introspect its tools. When it couldn't scan OptimEngine, I went looking for the reason, and the reason was an RFC 8414 compliance issue between how ScaleKit advertises itself and how Smithery expects authorization servers to be described.

The short version: ScaleKit's resource-scoped OAuth metadata correctly supports DCR (what Smithery needs), but its `issuer` claim points to the base environment URL rather than the resource-scoped URL. Smithery, strictly following RFC 8414, validates that `issuer` equals the `authorization_servers` URL declared in `/.well-known/oauth-protected-resource`. The mismatch fails the handshake.

I tried three fixes over a single Saturday evening, in three successive branches.

1. **Use the base URL as authorization server.** This matches the issuer, but the base URL doesn't expose DCR — Smithery still fails, now with a different error: "does not support dynamic client registration."

2. **Use the resource-scoped URL.** This has DCR but mismatched issuer. Back to square one.

3. **Serve the metadata myself.** Implement `/.well-known/oauth-authorization-server` as a proxy endpoint: fetch ScaleKit's real metadata, override the `issuer` field to match my own `BASE_URL`, return the rewritten document. All the OAuth flows still terminate at ScaleKit (authorize, token, register, jwks) — I only rewrite the one field Smithery validates. Works. RFC-compliant. Both compliance and DCR satisfied.

Even after this, Smithery's MCP scanner was still returning 401 during the handshake for unrelated reasons. Rather than keep debugging, I implemented a `/.well-known/mcp/server-card.json` endpoint — a static JSON document describing my tools — which Smithery docs explicitly support as a fallback for servers whose dynamic discovery fails. Paste the tool manifest, done, scanned.

**The lesson, if there is one**: in a young protocol ecosystem, working around integration quirks is sometimes faster than fixing them. Both OAuth spec compliance and Smithery's scanner are reasonable in isolation, but they didn't meet cleanly at the time. A metadata proxy plus a static fallback cost a few hours. Solving it "properly" at the ScaleKit layer would have required either them changing their issuer convention or me running my own OAuth provider — not an acceptable trade.

## What I'd do differently

Writing this up forces a kind of retrospective honesty. A few things I'd change if I started again:

**I'd build one domain deep before eight domains wide.** Eight optimization verticals in two months is impressive on paper. In practice, scheduling and routing are the only ones with real traffic; the others are options I left on the table. If the first use case is SME manufacturing, then the first version should do scheduling with deep configuration (shift windows, setup times, quality yield) and nothing else. Breadth was a hedge that slowed everything down.

**I'd wait on OAuth.** I built the OAuth 2.1 integration before I had a single paying customer who needed it. The open `/mcp` tier, with rate limiting, would have covered all real usage for the first ninety days. Adding auth earlier meant debugging ScaleKit-Smithery integration when I should have been writing articles or talking to users.

**I'd use one branch per problem, not per attempt.** The Smithery saga left orphan branches in my repo — each representing one attempted fix. Iterating on a single branch with `git commit --amend` or `git rebase -i` produces the same outcome with a cleaner history. The reason I didn't is psychological: new branches felt like "resets" during stressful debugging. Useful insight, next time.

**I'd start writing the same day I started the product.** The technical work is only half of what gets a developer tool adopted. Without an article trail describing decisions and lessons, nobody finds the work. I'm writing this article two months late. If you're shipping something similar, start writing about it from commit one.

## Try it yourself

OptimEngine is live. For Claude Desktop or any MCP client, add this to your config:

```json
{
  "mcpServers": {
    "optimengine": {
      "transport": "sse",
      "url": "https://optim-engine-production.up.railway.app/mcp"
    }
  }
}
```

Restart your client and the solver tools should appear in the tool list, ready to be called for scheduling, routing, packing and the other optimization domains. Free tier: ten calls per hour per IP, no authentication.

REST endpoints exist for server-to-server integration, but they're behind an API key — this article focuses on the MCP side, which is the open surface. Production tier with OAuth and higher limits is billed via x402 on Base — I'll cover the monetization layer in a separate article.

Source on GitHub at [github.com/MicheleCampi/optim-engine](https://github.com/MicheleCampi/optim-engine). If you're working on something similar — wrapping a scientific or domain-specific tool as an agent-accessible service — I'd genuinely like to hear about it. Reach out on X at [@MicheleC54474](https://twitter.com/MicheleC54474).
