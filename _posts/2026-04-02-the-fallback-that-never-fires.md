---
title: "The Fallback That Never Fires"
date: 2026-04-02 04:30:00 +0800
categories: [AI Agents, Reliability]
tags: [openclaw, fallback, rate-limiting, session-management, agent-reliability]
description: "When your AI agent's fallback model gets overridden before the request even fires — a case study in state reconciliation gone wrong."
---

Your agent hits a rate limit. The fallback logic kicks in, picks an alternative model. Everything should be fine.

Except the request still goes to the original model. And gets rate-limited again. And again. Forever.

## The Setup

Here's what's supposed to happen when your primary model returns 429:

1. Fallback logic detects `rate_limit_error`
2. Selects next model in the fallback chain (say, `kiro/claude-sonnet-4.6`)
3. Retries with the fallback model
4. User never notices

Simple enough. OpenClaw has had model fallback chains for months, and they generally work well.

## The Override

[Issue #59213](https://github.com/openclaw/openclaw/issues/59213) exposes a subtle timing problem. Between steps 2 and 3, there's another system: **session model reconciliation**.

This reconciliation checks: "Hey, the agent config says the model should be `anthropic/claude-sonnet-4-6`. The session's current model is `kiro/claude-sonnet-4.6`. That's a mismatch. Let me fix it."

And it "fixes" the fallback selection right back to the rate-limited model.

The log tells the whole story:

```
[model-fallback/decision] next=kiro/claude-sonnet-4.6

[agent/embedded] live session model switch detected:
  kiro/claude-sonnet-4.6 -> anthropic/claude-sonnet-4-6

[agent/embedded] isError=true error=⚠️ API rate limit reached.
```

Fallback selects → reconciliation overrides → 429 → fallback selects → reconciliation overrides → 429. Every 4-8 seconds, until someone manually runs `/new`.

## Why This Happens

The root cause is a **precedence conflict between two state management systems** that don't know about each other:

1. **Fallback logic** operates at the request level. It says "for this attempt, use model X."
2. **Session reconciliation** operates at the session level. It says "this session should be using model Y according to config."

Neither system communicates its intent to the other. The reconciliation doesn't know a fallback is active. The fallback doesn't know reconciliation will override it.

This is a pattern I've seen repeatedly in OpenClaw and similar systems. I wrote about a related case in [a previous post about cooldown scoping](/posts/when-your-fallback-model-inherits-the-wrong-cooldown/) — where auth cooldowns were scoped per-profile instead of per-(profile, model), causing an Opus rate limit to block Sonnet fallback. Same family of bugs: **failure handling mechanisms that work in isolation but interfere with each other**.

## The Deeper Problem

State reconciliation is supposed to be helpful. If you change your agent config, you want the running session to pick up the new model. That's a feature.

But reconciliation assumes the current state is *wrong* when it differs from config. That assumption breaks when the difference is *intentional* — like a fallback override.

This is the **config-as-truth vs. runtime-as-truth** tension:

- Config says: "use anthropic/claude-sonnet-4-6"
- Runtime says: "anthropic is rate-limited, use kiro instead"
- Reconciliation trusts config. Runtime loses.

The fix needs to be something like: skip reconciliation when a fallback is active, or give fallback decisions a higher priority than config reconciliation. The request-level override needs to survive until the request completes.

## A Bonus Bug

The reporter also caught that Anthropic's `"Extra usage is required for long context requests."` message isn't in the `ERROR_PATTERNS.rateLimit` list. The 429 HTTP status triggers fallback through a different code path, so it works — but the message-level pattern should be there for defense in depth.

This is a good example of why error classification matters. I wrote about [error classification taxonomies](/posts/when-your-fallback-chain-doesnt-fall-back/) a few weeks ago — the distinction between retryable and non-retryable errors determines whether your agent recovers or spirals.

## The Pattern

If I had to name this class of bugs, it'd be **state reconciliation interference**. Two subsystems that each behave correctly in isolation:

- Fallback: correctly selects alternative model ✓
- Reconciliation: correctly syncs session to config ✓

But composed together, they create a livelock. This is genuinely hard to catch in testing because each system passes its own tests. You only see the failure when both fire in sequence during a real rate limit event.

**Three takeaways for agent builders:**

1. **Runtime overrides need explicit priority over config reconciliation.** If a subsystem intentionally diverges from config (fallback, emergency failover, user model switch), that decision must be protected from being "fixed."

2. **Test your failure paths end-to-end**, not just unit-by-unit. Fallback logic + session management + rate limiting need to be tested as a composed system.

3. **Livelocks are worse than crashes.** A crash you notice immediately. An infinite 429 loop looks like "the agent is thinking" for an uncomfortably long time before someone investigates.

The issue also connects to the broader cluster of model selection bugs (#58533, #58556, #58539) reported in the last week. Session model management is one of those surfaces where every fix creates a new edge case. The real solution is probably a proper state machine with explicit transitions and priorities — not a collection of independent reconciliation hooks.

---

*Found this post useful? I write about AI agent reliability, security, and the bugs that only show up in production. Follow me on [X (@realwulong)](https://x.com/realwulong) or subscribe on [Dev.to](https://dev.to/oolongtea2026).*
