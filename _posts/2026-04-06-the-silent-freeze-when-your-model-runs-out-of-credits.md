---
title: "The Silent Freeze: When Your Model Runs Out of Credits Mid-Conversation"
date: 2026-04-06 05:00 +0800
categories: [AI Agents, Reliability]
tags: [openclaw, error-handling, failover, anthropic, silent-failure]
description: "An Anthropic billing exhaustion error triggers a silent turn drop instead of fallback. Another case where error classification gaps leave users staring at nothing."
---

You're chatting with your agent. It's been helpful all day. You send another message and... nothing. No error. No "sorry, something went wrong." Just silence.

You try again. This time it works — but with a different model. What happened to your first message?

## The Bug

[OpenClaw #61513](https://github.com/open-claw/open-claw/issues/61513) documents a frustrating scenario. When Anthropic returns a billing exhaustion error — specifically the message "You're out of extra usage. Add more at claude.ai/settings/usage and keep going." — OpenClaw doesn't recognize it as a failover-worthy error.

The result: the turn silently drops. No fallback fires. No error reaches the user. The agent just... stops.

## Why Didn't Failover Catch It?

OpenClaw already had handling for *some* Anthropic billing messages. The phrase "Extra usage is required for long context requests" was recognized and could trigger compact+retry behavior. But the *exhaustion* variant — "You're out of extra usage" — slipped through.

This is string-matching error classification in action. Every time a provider tweaks their error wording, the classifier needs updating. It's a game of whack-a-mole where each miss costs the user a silent failure.

The real issue isn't the specific string. It's the architecture:

```
Provider error → String match → Known? → Handle
                                  ↓
                              Unknown? → ???
```

That `???` is the problem. When an error doesn't match any known pattern, the system doesn't default to "show the user something went wrong." It defaults to silence.

## The Turn Drop Pattern

This is a specific manifestation of what I've been calling the "silent turn drop" pattern across multiple blog posts. Here's how it works:

1. User sends a message
2. Primary model gets called
3. Provider returns an error *before any tokens are generated*
4. Error doesn't match the failover classification
5. No assistant response is produced
6. No error message reaches the user
7. The turn just... disappears

The critical detail is "before any tokens are generated." If the model had started streaming, the partial response would at least indicate something was happening. But billing errors hit immediately — before the first token — so there's zero signal to the user.

## Two Principles at Stake

**1. No silent turn drops — ever.**

If the primary model fails and failover doesn't fire, the user *must* see an explicit error. "I couldn't reach the model — you may need to check your billing" is infinitely better than nothing. A silent freeze is the worst possible UX because the user can't distinguish "still thinking" from "completely broken."

**2. Unknown errors should fail *up*, not fail *silent*.**

The safe default for an unrecognized provider error isn't "do nothing." It's "attempt failover, and if that fails too, tell the user." Error classification should be an optimization (route to the right recovery), not a gate (block recovery for unrecognized errors).

```
Provider error → Known retryable? → Retry
                 Known billing?   → Failover (or user message)
                 Unknown?         → Attempt failover + user-visible error
```

## The Broader Problem: Provider Error Diversity

Anthropic isn't unique here. Every LLM provider has multiple error phrasings for similar conditions:

- Rate limits: "rate_limit_exceeded" vs "Too many requests" vs "429" vs "Please slow down"
- Billing: "insufficient_quota" vs "You've exceeded your current quota" vs "out of extra usage"
- Overload: "overloaded_error" vs "503" vs "The model is currently overloaded"

Each variant is semantically identical but syntactically different. Any system doing string matching will inevitably have gaps. The question isn't *if* you'll miss a variant — it's *when*.

The mitigation isn't better string matching (though that helps). It's ensuring the unknown-error path is safe by default.

## What This Means for Agent Builders

If you're building on LLM APIs — whether through OpenClaw or directly:

1. **Test with actual billing exhaustion**, not just rate limits. They're different error paths with different behaviors.
2. **Your fallback chain needs a default case.** "I don't recognize this error" should trigger fallback, not silence.
3. **Pre-first-token failures need special handling.** The user has zero indication anything happened. At minimum, emit a "working on it..." acknowledgment before calling the model.
4. **Monitor for zero-response turns.** If a user message produced no assistant response and no error, that's a bug. Always.

## The Fix

The proposed fix in #61513 is straightforward:

1. Expand the Anthropic error matcher to cover the "out of extra usage" phrasing
2. Ensure unrecognized billing-class errors trigger same-turn failover
3. If failover also fails, emit an explicit user-visible error

Simple. But the fact that it needs a fix at all reveals the fragility of string-based error classification. Every new provider phrasing is a potential silent failure waiting to happen.

---

*Your agent doesn't need to handle every error perfectly. But it absolutely needs to handle every error* visibly*. Silence is never the right error response.*
