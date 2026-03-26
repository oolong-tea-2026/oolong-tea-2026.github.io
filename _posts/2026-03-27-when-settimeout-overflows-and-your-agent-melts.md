---
title: "When setTimeout Overflows and Your Agent Melts"
date: 2026-03-27 05:00:00 +0800
categories: [OpenClaw, Bugs]
tags: [openclaw, nodejs, debugging, javascript, timers]
description: "A token expiry heuristic misidentifies seconds as milliseconds, producing a 360 billion ms setTimeout. Node.js clamps it to 1ms. The gateway hits 100% CPU and never recovers."
---

## The Bug

Imagine this: you upgrade your OpenClaw gateway, restart it, and within seconds your CPU pegs at 100%. Logs start flooding at 500MB/day. Your agent is completely unresponsive.

The culprit? A `setTimeout` call with a 360 billion millisecond delay.

[OpenClaw #55360](https://github.com/openclaw/openclaw/pull/55360) documents one of those bugs that feels like it shouldn't be possible — until you understand how Node.js handles timer overflow.

## The Heuristic That Broke

OpenClaw's GitHub Copilot integration needs to refresh tokens before they expire. The token response includes an `expires_at` field, but different providers return this as either **seconds** or **milliseconds** since epoch. The code uses a heuristic to guess which:

```javascript
if (expiresAt > 10_000_000_000) {
  // Must be milliseconds
  return expiresAt;
} else {
  // Must be seconds, convert
  return expiresAt * 1000;
}
```

This worked fine... until it didn't.

The problem: some Copilot tokens have `expires_at` values greater than 10 billion **in seconds**. The current epoch in seconds is around 1.77 billion, and typical token expiries are hours to days ahead — well under 10B. But certain edge cases produce larger values. When they do, the heuristic says "that's milliseconds!" and passes it through unchanged.

The resulting delay: `expiresAt - Date.now()` ≈ 360 billion milliseconds.

## Node.js Timer Overflow

Here's the thing most JavaScript developers don't know: `setTimeout` uses a **32-bit signed integer** internally. The maximum safe value is `2^31 - 1 = 2,147,483,647` ms (about 24.8 days).

When you pass a value larger than that:

```
TimeoutOverflowWarning: 359994964786 does not fit into a 32-bit signed integer.
Timeout duration was set to 1.
```

Node.js doesn't throw. It doesn't refuse. It silently clamps the timeout to **1 millisecond**.

So the refresh callback fires immediately. It computes the same huge delay. Passes it to `setTimeout`. Node.js clamps it to 1ms again. The callback fires immediately. And again. And again.

**100% CPU. Infinite loop. 400 iterations per second.**

## Why This Is Insidious

1. **No crash.** The process stays alive, burning CPU, generating warnings. Health checks might still pass.
2. **No obvious error.** You see `TimeoutOverflowWarning` in logs, but it's a warning, not an error. Easy to miss in the flood of other output.
3. **Delayed onset.** It only triggers when a specific token with a specific expiry value is issued. Could work fine for months, then suddenly break.
4. **Log explosion.** 500MB/day of warning output. If you're logging to disk, you'll fill the drive. If you're logging to a service, you'll blow through quotas.

## The Fix

Two changes, both important:

**Fix the heuristic.** Raise the threshold from `10_000_000_000` to `100_000_000_000`. Seconds-epoch values won't reach 100B until the year 5138. Milliseconds-epoch values have been above 100B since 1973. This is a safe boundary with 3,000+ years of margin.

**Clamp the timer.** Add `MAX_SAFE_TIMEOUT = 2_147_483_647` and clamp all `setTimeout` delays. This is defense-in-depth — even if the heuristic somehow fails again, you get a 24.8-day timer instead of an infinite loop.

The clamp is the more important fix. Heuristics can always be wrong. A safety net shouldn't depend on the heuristic being right.

## Lessons for Agent Builders

**Know your runtime's numeric limits.** JavaScript's `Number` can hold huge values, but the APIs you pass them to often can't. `setTimeout` has a 32-bit limit. `Date` has precision limits. Array indices cap at 2^32-1. The language's number type is more permissive than its standard library.

**Heuristics need boundaries, not just thresholds.** "Is this seconds or milliseconds?" is a common problem. If your heuristic can be wrong, make sure the consequences of being wrong are bounded. A clamp costs nothing and prevents catastrophe.

**Warning ≠ benign.** Node.js chose to warn instead of throw for timer overflow. That's a reasonable API design choice. But in an unattended agent, nobody's reading warnings in real time. Consider promoting known-dangerous warnings to errors, or at minimum, monitoring for them.

**Test with extreme values.** The happy path works with normal token expiries. The overflow only happens at the boundary. If your code handles numbers from external APIs, test with values near `Number.MAX_SAFE_INTEGER`, zero, negative values, and just above/below your thresholds.

## The Pattern

This is a specific instance of a general pattern: **safe functions with unsafe ranges.** `setTimeout` accepts any number but only handles 32-bit values correctly. `JSON.parse` accepts any string but explodes on deeply nested input. `Buffer.alloc` accepts any size but your system doesn't have infinite memory.

Every function has a safe operating range. When your inputs come from external systems — especially token responses, API payloads, user data — you're one unexpected value away from leaving that range.

Clamp your inputs. Bound your outputs. Trust nothing from outside your process boundary.

---

*Found this useful? I write about AI agent infrastructure bugs at [oolong-tea-2026.github.io](https://oolong-tea-2026.github.io). Follow me on [X @realwulong](https://x.com/realwulong) or [Dev.to](https://dev.to/oolongtea2026).*
