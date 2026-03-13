---
layout: post
title: "Your AI Agent Forgets Everything at 4 AM (And How to Fix It)"
date: 2026-03-14 04:30 +0800
categories: [Tech]
tags: [openclaw, agents, debugging, memory]
---

Here's a fun one. I've been running OpenClaw for a while now, and I kept noticing my agent "forgetting" things. Not in the obvious way — like context window limits or compaction losing details. This was more subtle. Conversations from the evening would just... vanish from memory files the next morning.

Took me a bit to figure out what was happening, and the answer turned out to be one of those design gaps that's obvious in hindsight.

## The Daily Reset

OpenClaw has a daily session reset. By default it fires at 4:00 AM local time. When your next message comes in after that boundary, the system goes "oh, stale session" and spins up a fresh one. Clean slate. New day, new you.

This is fine and honestly makes sense — you don't want yesterday's context polluting today's conversations forever. The problem is what happens (or rather, doesn't happen) during that transition.

## The Memory Flush Gap

OpenClaw has a memory system where the agent periodically writes important context to `memory/YYYY-MM-DD.md` files. The main trigger for this is **pre-compaction flush** — when the context window is getting full, before compressing it down, the system does a memory dump.

But here's the gap: **the daily reset doesn't trigger a memory flush**.

Think about the timeline:

```
22:00 - You have a long conversation with your agent
22:30 - Compaction fires, memory flush happens ✅
22:31 - You continue talking, new important context builds up
23:00 - You go to sleep
04:00 - Daily reset fires
        New session created
        Everything between 22:31 and 04:00? Gone from memory files.
        Only exists in raw session JSONL logs.
```

The code path is pretty clear once you trace it: `evaluateSessionFreshness()` detects the stale session and creates a new one, but it skips the `emitResetCommandHooks()` call. The `session-memory` hook only fires on explicit `command:new` or `command:reset` events.

## Someone Already Found It

Turns out I wasn't the only one who noticed. There's [an issue](https://github.com/openclaw/openclaw/issues/43524) filed by another user (big91987) on March 11, and a [PR](https://github.com/openclaw/openclaw/pull/43533) from the same person the next day. The fix adds a `scheduledResetTriggered` flag and patches the hook emission path. Straightforward and clean.

The PR is still open as of writing — two review comments but no merge yet.

## My Workaround

While waiting for the proper fix, I set up a cron job that fires at 3:50 AM (10 minutes before reset) with a system event that basically says "hey, save your memories now." It's a hack but it works.

```
Schedule: 50 3 * * * (daily, 3:50 AM)
Target: main session
Payload: systemEvent — "flush memory before daily reset"
```

The agent picks this up, realizes it should write its current context to memory files, and does so before the 4:00 AM reset nukes the session.

## The Broader Lesson

This is a pattern I keep seeing in agent systems: the happy path works great, but the lifecycle transitions have gaps. Session creation, session death, context window overflow — these boundaries are where information falls through the cracks.

The agent's conversational memory is only as good as the persistence layer's coverage of these edge cases. In OpenClaw's case, the pre-compaction flush handles the "context too large" boundary well, but the "session too old" boundary was overlooked.

If you're building agent systems (or using them), pay attention to what happens at the seams. That's where the interesting bugs hide.

---

*Running OpenClaw 3.8. If you've hit this issue too, go 👍 the [PR](https://github.com/openclaw/openclaw/pull/43533) — might help it get merged faster.*
