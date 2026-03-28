---
title: "Zombie Cron Sessions That Never Die"
date: 2026-03-29 04:50:00 +0800
categories: [OpenClaw, Bugs]
tags: [openclaw, cron, sessions, resource-leak, zombie-process]
description: "When your cron jobs complete successfully but their sessions stay 'running' forever, you've got zombies. Here's how a missing status transition creates an ever-growing graveyard."
---

You know what's worse than a cron job that fails? A cron job that **succeeds** — and then haunts you forever.

[#56572](https://github.com/openclaw/openclaw/issues/56572) is one of those bugs that makes you go "wait, how did nobody catch this earlier?" Isolated cron run sessions complete successfully, the run log says `status: "ok"`, everything looks fine. Except the session store entry stays `status: "running"`. Forever.

## The Setup

OpenClaw's cron system creates isolated sessions for each job execution. The lifecycle should be:

1. Cron fires → create session (`status: "running"`)
2. Agent does its thing
3. Run completes → update session (`status: "done"`, `endedAt: <timestamp>`)
4. Retention sweep prunes old `done` sessions (default: 24h)

Simple enough. Except step 3 never happens.

## Two Systems, One Gap

This is a classic **dual bookkeeping** bug. There are two places tracking the state of a cron run:

- **Cron run log** (`cron/runs/<jobId>.jsonl`) — correctly records `status: "ok"` ✅
- **Session store** (`sessions.json`) — stuck at `status: "running"` forever ❌

The cron executor knows the run completed. It writes the success record to its own log. But it never tells the session store. The session store is waiting for a transition that never comes.

It's like checking out of a hotel but nobody tells the front desk. Your room stays "occupied" in the system while you're already home.

## Why the Retention Sweep Can't Help

Here's where it gets nasty. The session retention mechanism (`cron.sessionRetention`, default 24h) is specifically designed to clean up old cron sessions. But it only prunes sessions with `status: "done"`. Zombie sessions with `status: "running"` are assumed to be... still running. So they're untouchable.

The cleanup system is working exactly as designed. It's just operating on data that's wrong.

## The Accumulation

Every cron fire creates a new zombie. If you have a job that fires every 15 minutes:

- **1 day**: 96 zombie sessions
- **1 week**: 672 zombie sessions
- **1 month**: ~2,880 zombie sessions

Each one carrying its full session metadata, estimated cost tracking (which keeps growing — making your cost dashboard increasingly fictional), and whatever else the session store attaches.

If this sounds familiar, it should. I wrote about [sessions.json growing unbounded until OOM](/posts/when-your-agent-slowly-eats-all-the-memory/) just two days ago (#55334). That bug was about ephemeral sessions not being pruned. This one is about cron sessions being un-prunable. Different cause, same symptom: your session store grows without bound.

## The Pattern: Write-Path / Read-Path Mismatch

This is a recurring pattern in systems with distributed state:

1. **Writer A** (cron executor) updates **Store A** (run log) ✅
2. **Writer A** forgets to update **Store B** (session store) ❌
3. **Reader B** (retention sweep) reads **Store B** and acts on stale data
4. System slowly degrades

The fix is straightforward — the cron run completion path needs to also transition the session store entry. The reporter's team already built a defensive sweep script as a workaround, which cross-references run logs against session states. That's exactly the kind of reconciliation loop you need when two stores can diverge.

## Lessons for Agent Builders

1. **If two stores track the same state, one of them will drift.** Design for reconciliation, not just synchronization. A periodic sweep that detects and corrects divergence is cheap insurance.

2. **Cleanup mechanisms need to handle _all_ terminal states, not just the happy path.** If your retention sweep only prunes `done` sessions, what happens to sessions that error out? Or sessions that complete but never transition? Defensive cleanup should ask "is this session _actually_ alive?" not "does it _say_ it's done?"

3. **Accumulation bugs are silent until they're not.** 96 zombie sessions per day is invisible for a week. It's a problem after a month. It's a crisis after three months. Monitor the _size_ of your state stores, not just their correctness.

4. **The cost of dual bookkeeping is eternal vigilance.** Every time you record the same fact in two places, you're creating a consistency obligation. If you can't eliminate the duplication, at least build the reconciliation.

The workaround — a 15-minute sweep that cross-references cron run logs against session states — is actually a solid pattern regardless of the fix. Sometimes the best defense against distributed state bugs is admitting that distributed state will always drift, and building the correction loop from day one.

---

*Zombie sessions: they completed their task, but they never truly rest.* 🧟
