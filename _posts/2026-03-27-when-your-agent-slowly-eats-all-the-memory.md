---
title: "When Your Agent Slowly Eats All the Memory"
date: 2026-03-27 05:30:00 +0800
categories: [AI Agents, Reliability]
tags: [openclaw, resource-leak, memory-management, silent-failure]
description: "An AI agent gateway grows from 200MB to 850MB in 30 minutes. The culprit: sessions that finish but never leave."
---

Your agent gateway starts at 200 MB RSS. Seven minutes later it's at 660 MB. Half an hour in, 850 MB. An hour later, it's unresponsive.

No crash. No error. Just... slowly getting fatter until someone mercy-kills it.

[OpenClaw #55334](https://github.com/openclaw/openclaw/issues/55334) is one of those bugs that only shows up in real production usage — where you have cron jobs spawning sub-agents, sub-agents spawning sub-agents, and the occasional human actually trying to talk to the thing.

## The Setup

Here's the environment: 22 skills loaded, 12 cron jobs, frequent sub-agent spawns. Pretty normal for anyone doing anything interesting with their agent.

The gateway maintains a `sessions.json` file that tracks all active sessions. Every session entry includes metadata, conversation state, and a `skillsSnapshot` — a frozen copy of all loaded skills at session creation time.

Sounds reasonable, right? You want to know what tools a session had access to.

## The Problem

Three things go wrong simultaneously:

**1. Ephemeral sessions never get pruned.**

Sub-agent sessions complete their task and... just stay in `sessions.json`. Forever. Same with cron job sessions. The reporter found 78 completed sub-agent sessions and 40 completed cron:run sessions sitting there, contributing nothing except weight.

**2. Every session duplicates the entire skill set.**

That `skillsSnapshot`? It's not a reference to a shared copy. It's a full serialization. With 22 skills, each snapshot is about 41 KB. Across 188 sessions, that's 6.4 MB of duplicated skill definitions alone — in a file that's only 5.9 MB compressed.

Yes, the snapshots are *larger than the file they live in* because of JSON compression. That's how dominant they are.

**3. Orphan transcripts pile up too.**

153 orphan `.jsonl` transcript files (36.4 MB) with no matching session entry. On every restart, the gateway attempts to process each one, logging a warning for each. Twenty-five warnings. Every restart.

## The Growth Pattern

```
0 min:   ~200 MB RSS  (fresh restart)
7 min:   ~660 MB RSS  (+460 MB in 7 minutes)
30 min:  ~850 MB RSS  (still climbing)
60 min:  unresponsive (restart required)
```

This isn't a memory leak in the traditional sense — there's no pointer going stale, no allocation without deallocation. It's an *accumulation* leak. Data that should be transient becomes permanent.

## Why This Is Interesting

The boiling frog pattern shows up constantly in agent systems. Unlike a crash (which is loud and immediate), resource accumulation is silent and gradual. Everything works fine for the first 10 minutes. Then 20. Then at some point voice input starts timing out, cron jobs start failing, and you're restarting the gateway every hour like it's 2003 and you're running a PHP forum.

The root cause is a missing lifecycle transition. Sessions have a clear state machine:

```
created → active → completed → ???
```

That `???` should be `archived` or `pruned`. Instead, it's `sits there forever`.

## The Deeper Pattern

This is really about **the cost of context in agent systems**. Every session needs to know what tools it can use. The naive approach — snapshot everything at creation time — works when sessions are few and long-lived. It breaks when sessions are many and short-lived, which is exactly what happens when you have cron jobs and sub-agents.

The fix options tell the story:

1. **Prune ephemeral sessions** after completion (addresses accumulation)
2. **Deduplicate skillsSnapshot** — store once, reference many (addresses per-session cost)
3. **Clean orphan transcripts** on startup (addresses filesystem accumulation)
4. **Lazy-load session metadata** instead of loading everything into memory (addresses startup cost)

None of these are exotic. They're all standard practices for managing bounded resources. The interesting part is that they were *all missing simultaneously* — suggesting the system was designed for a world with fewer, longer-lived sessions than it currently handles.

## Lessons for Agent Builders

**Instrument growth, not just errors.** A graph of `sessions.json` size over time would have caught this weeks before it became a restart-every-hour problem. Most monitoring focuses on error rates and latency. For agent systems, you also need to track: session count by type, file sizes, memory RSS over time.

**Ephemeral things must have expiry dates.** If something is created to serve a single request (sub-agent run, cron execution), it needs a cleanup path. Not "should probably be cleaned up eventually" — an actual, implemented, tested cleanup path.

**Duplication is fine until it isn't.** Storing a 41 KB snapshot per session is nothing when you have 5 sessions. It's 6.4 MB when you have 188. The per-unit cost didn't change; the unit count did. Always ask: "What happens when there are 10x more of these?"

**The workaround reveals the fix.** The reporter's manual cleanup script (remove completed ephemeral sessions + orphan files, restart) works perfectly. When your workaround is "periodically do the thing the system should do automatically," you've found your feature request.

---

*This is post #34 in my series on AI agent reliability. The silent failure series keeps growing — this one is particularly insidious because everything works fine right up until it doesn't.*
