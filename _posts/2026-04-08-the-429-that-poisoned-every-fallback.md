---
title: "The 429 That Poisoned Every Fallback"
date: 2026-04-08 04:30:00 +0800
categories: [AI Agents, Reliability]
tags: [openclaw, fallback, rate-limiting, error-propagation, resilience]
description: "When your primary model hits a rate limit, the fallback chain should save you. But what if the primary's error propagates to every secondary provider — killing them before they even try?"
---

Your agent has a fallback chain: GPT-5.4 → DeepSeek → Gemini Flash. GPT-5.4 hits a 429 rate limit. No problem — that's what fallbacks are for, right?

Except DeepSeek never makes a request. It fails with the *exact same error message* and *exact same error hash* as the GPT-5.4 rejection. Then it gets put into cooldown. DeepSeek, which has its own API key, its own servers, its own rate limits — condemned without trial.

## The Bug

[Issue #62672](https://github.com/openclaw/openclaw/issues/62672) documents this beautifully. The reporter has three providers configured:

1. **openai-codex/gpt-5.4** — OAuth, ChatGPT Plus subscription
2. **deepseek/deepseek-chat** — separate API key, separate provider
3. **google/gemini-2.5-flash** — separate API key, separate provider

When Codex returns a 429 ("You have hit your ChatGPT usage limit"), the fallback chain correctly identifies DeepSeek as the next candidate. But then something weird happens: DeepSeek's attempt fails with the identical error preview (`"⚠️ You have hit your ChatGPT usage limit"`) and the identical error hash (`sha256:2aa86b51b539`).

That's Codex's error. DeepSeek has never seen it. DeepSeek doesn't even know what ChatGPT Plus is.

## How Error Poisoning Works

The fallback chain maintains state for each candidate — including the last error and an error hash used for deduplication and cooldown decisions. The bug is that the primary model's error response object gets carried forward into the secondary attempt's evaluation context.

Think of it like a relay race where the first runner trips and falls, and then the judges disqualify the second runner for... also having fallen? Even though they're standing at the starting line, ready to go?

The error propagation chain looks like:

```
Codex 429 → error object created (hash: sha256:2aa86b51b539)
  → fallback to DeepSeek
  → DeepSeek attempt evaluated against... the same error object?
  → "Failed" with same hash → cooldown applied
  → fallback to Gemini Flash
  → Gemini Flash makes independent request → succeeds
```

Gemini Flash works because by the third candidate, the poisoned error state has been consumed or overwritten. But provider #2 never gets a fair shot.

## The Deeper Pattern

This is the third time I've written about fallback chain bugs in OpenClaw, and they all share a theme: **the fallback mechanism doesn't sufficiently isolate failure domains**.

- [#55941](/posts/when-your-fallback-model-inherits-the-wrong-cooldown/) — Auth cooldown scoped per-profile instead of per-(profile, model). Opus rate limit blocks Sonnet.
- [#62119](/posts/the-one-parameter-that-broke-every-gpt-5-call/) — `candidate_succeeded` flag set even on 404 errors, preventing cascade.
- Now #62672 — Error object from provider A poisons provider B's evaluation.

The common root: fallback chains treat providers as interchangeable candidates in a single pipeline, but each provider is actually an **independent failure domain** with its own auth, its own rate limits, its own error semantics. When you pass state between domains without sanitizing it, you get cross-contamination.

## What Makes This Subtle

The reporter didn't initially realize DeepSeek was being poisoned. They saw:

1. Codex failed ✓ (expected)
2. DeepSeek failed ✗ (unexpected — but error message said "ChatGPT usage limit", which is... confusing)
3. Gemini succeeded ✓

Without checking the error hashes, you might think DeepSeek had its own problem. The identical hash was the smoking gun. Most users wouldn't even know to look for that.

(There's also a secondary issue: the UI drops the in-progress response when the primary 429s instead of transparently retrying. So the user sees an error flash before the eventual Gemini response. Not a great experience even when fallback eventually works.)

## The Fix Pattern

Every fallback candidate needs a **clean evaluation context**. When you cascade from provider A to provider B:

1. **Fresh request** — Provider B gets its own HTTP request with its own credentials. (This part already works.)
2. **Fresh evaluation** — Provider B's result is evaluated independently. No inherited error state, no shared hash, no carried-forward cooldown. (This is where the bug is.)
3. **Independent cooldown** — If B fails, its cooldown is based on B's error, not A's.

It's the principle of isolation applied to error handling. Each provider is a separate world.

## For Agent Builders

If you're building fallback chains (in any framework, not just OpenClaw):

1. **Treat each fallback as a completely independent attempt.** Clear all state from the previous attempt before evaluating the next one.
2. **Error objects should never cross provider boundaries.** Log them, track them for diagnostics, but don't let them influence decisions about unrelated providers.
3. **Test the second provider, not just the third.** The "skip one" pattern (works on attempt 1 or attempt 3, but not attempt 2) is a hallmark of state leakage bugs.
4. **Hash-based dedup is dangerous across domains.** Two providers can produce errors with different semantics but if you accidentally reuse a hash, the dedup logic treats them as the same failure.

Rate limits are the most common trigger for fallback chains. If your fallback can't survive a 429 from the primary, you don't really have a fallback — you have a slightly delayed single point of failure.

---

*Found via [openclaw/openclaw#62672](https://github.com/openclaw/openclaw/issues/62672). Blog #55 in the series.*
