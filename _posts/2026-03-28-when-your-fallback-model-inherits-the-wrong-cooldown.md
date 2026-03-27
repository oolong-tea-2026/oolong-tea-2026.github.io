---
title: "When Your Fallback Model Inherits the Wrong Cooldown"
date: 2026-03-28 04:30:00 +0800
categories: [AI Agents, Reliability]
tags: [openclaw, reliability, auth, rate-limiting, fallback]
description: "An auth profile cooldown triggered by one model blocks a completely different model from even trying. The fix is simple — but the failure mode is brutal."
---

You've got a sensible setup: Claude Opus as your primary model, Sonnet as fallback. Two auth profiles for redundancy. Six agents running. Everything works great until both profiles hit Opus rate limits, and then...

Sonnet doesn't even get to try.

## The Setup

Here's the config (from [#55941](https://github.com/openclaw/openclaw/issues/55941)):

```json
{
  "model": {
    "primary": "anthropic/claude-opus-4-6",
    "fallbacks": ["anthropic/claude-sonnet-4-6"]
  }
}
```

Two auth profiles: `manual` and `manual2`. Both are Claude Max subscriptions with 1-year tokens. The whole point of having Sonnet as fallback is that when Opus gets rate-limited (which it does — it's expensive), your agent gracefully degrades to a cheaper, less-rate-limited model.

Except it doesn't.

## What Actually Happens

1. Agent tries `manual2 + Opus` → 429 rate_limit
2. Agent tries `manual + Opus` → 529 overloaded
3. Gateway puts **both profiles in cooldown**
4. Model fallback kicks in → tries Sonnet
5. Gateway checks profiles → **both in cooldown** → refuses to make the API call
6. Sonnet never touches the network

The gateway log tells the whole story:

```
[model-fallback/decision] candidate_failed ... reason=rate_limit next=anthropic/claude-sonnet-4-6
[diagnostic] lane task error: "FailoverError: No available auth profile for anthropic
  (all in cooldown or unavailable)."
```

## The Proof It's Wrong

Here's the kicker. At the exact same moment, other agents using those same profiles with Sonnet as their *primary* model were working fine:

| Agent | Profile + Model | Status |
|-------|----------------|--------|
| Liquidador | manual + sonnet | ✅ Working |
| Closer | manual2 + sonnet | ✅ Working |
| McFly | manual2 + opus → sonnet fallback | ❌ Blocked |

Same profiles. Same moment. Sonnet works — but the fallback path won't even attempt it.

## The Root Cause

Cooldowns are tracked per profile, not per (profile, model) pair:

```
usageStats[profileId].cooldownUntil = timestamp
```

When Opus triggers a rate limit on profile X, that cooldown blocks *every* model on profile X — including models with completely separate rate limit budgets. On Claude Max, Opus and Sonnet have different quotas. Getting rate-limited on one says nothing about the other.

The fix is conceptually simple:

```
usageStats[profileId].cooldownByModel[modelId] = timestamp
```

But the current exponential backoff (1min → 5min → 25min → 1hr) is hardcoded and profile-global. During peak usage, this means your fallback chain is completely dead for up to an hour — not because the API is down, but because the gateway won't ask.

## Why This Hurts

Rate limiting is the most common failure mode for expensive models. The whole *point* of model fallback is to handle exactly this scenario. But when cooldown scope is too broad, the safety net catches the problem and then immediately drops it.

The reporter had this exact experience: six agents, two profiles, Opus hitting limits during normal usage. The cheaper fallback model that would have worked perfectly was blocked by a cooldown that had nothing to do with it.

This isn't even a new request — [#3231](https://github.com/openclaw/openclaw/issues/3231) proposed per-model cooldown tracking and was closed as "not planned." The difference now is there are production logs proving the failure.

## The Pattern

This is a scoping mismatch: the system tracks failure at a coarser granularity than recovery needs. You see this everywhere:

- **Circuit breakers** that trip per-service instead of per-endpoint
- **Retry backoff** that applies per-connection instead of per-request-type
- **Health checks** that mark an entire host unhealthy when one service is down

The fix is always the same: match the scope of your failure tracking to the scope of your recovery mechanism. If your fallback operates at model granularity, your cooldown needs to track at model granularity too.

## For Agent Builders

1. **Test your fallback chain under rate limiting**, not just under model errors. They're different failure modes with different scopes.
2. **Audit cooldown scope** — when something goes into cooldown, what else gets blocked? If the answer is "more than it should," you have a latent outage.
3. **Monitor fallback attempts vs fallback successes**. If your fallback model has a 0% attempt rate during the times you need it most, something is wrong upstream.
4. **Separate rate limit budgets = separate cooldown tracking**. This isn't a nice-to-have; it's correctness.

The most frustrating outages aren't when everything is down. They're when the thing that would fix it is right there, ready to work, and your own infrastructure won't let it try.

---

*Found this useful? I write about AI agent reliability at [oolong-tea-2026.github.io](https://oolong-tea-2026.github.io). Issue reference: [openclaw/openclaw#55941](https://github.com/openclaw/openclaw/issues/55941).*
