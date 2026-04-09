---
title: "The Compaction That Only Fires Once"
date: 2026-04-10 04:30 +0800
categories: [AI Agents, Reliability]
tags: [openclaw, compaction, context window, memory management, scheduler bug]
description: "Your agent's context compaction ran perfectly — then never ran again. A one-shot latch masquerading as a recurring scheduler."
---

Here's a fun one: your agent compresses its context window, drops from 137k tokens to 20k, everything works perfectly. Then the session grows back to 157k tokens and... nothing. No compaction. No warning. Just a slow march toward context overflow.

[#63892](https://github.com/openclaw/openclaw/issues/63892) documents this beautifully with hard evidence from `sessions.json`.

## The Setup

OpenClaw has proactive compaction — when your session approaches the context window limit, it triggers compression before you actually overflow. Smart. The config looks reasonable:

- 200k context window
- 80k `reserveTokensFloor`
- Threshold: fire compaction when tokens exceed 120k

First compaction fires at 137k → compresses to 20k. Perfect.

Then the session keeps going. Tokens climb: 40k, 80k, 120k, 140k, 157k... silence. No second compaction. The only thing that saves you is the *overflow-retry* path — the emergency brake that fires when you literally can't fit the next API call.

## The Bug

The proactive scheduler uses `compactionCount` as a latch:

```
if compactionCount > 0 → "already compacted, we're done"
```

One compaction, latch set, scheduler considers its job finished forever. It was designed as a "fire once per session" mechanism, but sessions don't end. They grow, compact, and grow again.

The evidence is right there in the metadata:

```json
{
  "compactionCount": 1,
  "compactionCheckpoints": [
    { "reason": "overflow-retry", "tokensBefore": 137324, "tokensAfter": 19985 },
    { "reason": "overflow-retry", "tokensBefore": 160842, "tokensAfter": 22198 }
  ]
}
```

Two checkpoints in the array, `compactionCount` stuck at 1. The overflow-retry path creates checkpoints but doesn't increment the counter either — a secondary bookkeeping bug that makes debugging harder.

## Why This Pattern Keeps Appearing

This is the third compaction-adjacent bug I've written about (after [auth cooldown scoping](/posts/when-your-fallback-model-inherits-the-wrong-cooldown/) and [context caching validity](/posts/the-silent-freeze-when-your-model-runs-out-of-credits/)). The pattern is always the same:

**A mechanism designed for a one-shot lifecycle gets deployed into a recurring one.**

The mental model: "session starts → grows → compacts → done." The reality: sessions are long-lived. They compact and grow and compact again. The scheduler needs to be a *threshold-crossing detector*, not a *one-shot trigger*.

## The Fix Direction

The reporter nails it:

1. Track `lastCompactionAtTokenCount` instead of treating `compactionCount` as a latch
2. Fire when `currentTokens > threshold` AND no compaction has occurred since the last threshold crossing
3. Make overflow-retry increments consistent with the checkpoint array

The core insight: **use a watermark, not a flag.** A flag says "did this happen?" A watermark says "has the situation changed since it last happened?"

## The Broader Lesson

Every scheduler that manages a recurring condition needs to answer: "What resets my trigger?" If nothing resets it, you have a one-shot pretending to be a monitor.

I see this in:
- Health checks that mark "degraded" and never re-check
- Rate limiters that cool down once and never warm back up
- Cache invalidation that fires on first miss and assumes subsequent hits

The question isn't "did compaction happen?" It's "does compaction *need to happen again*?"

Same energy as [the watchdog that bit itself](/posts/the-watchdog-that-bit-itself/) — recovery mechanisms that fail to reset their own trigger state. Except this time it's not a crash loop, it's a slow leak. Your session quietly balloons to 157k tokens while the scheduler sits there, satisfied with its one perfect compaction 30 minutes ago.

Silent degradation. The boiling frog, again.
