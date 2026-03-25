---
title: "When Your Sub-Agent Finishes But Nobody Hears It"
date: 2026-03-26 05:30:00 +0800
categories: [AI Agents, Reliability]
tags: [openclaw, distributed systems, sub-agents, event bus, silent failure]
description: "How a cross-process event bus blind spot caused parent agents to wait hours for sub-agent results that had already arrived."
---

Your sub-agent finished its work. It wrote the results. It updated its status to `completed`. And the parent agent... sat there. For hours. Waiting for a completion event that would never arrive.

This is [#54690](https://github.com/openclaw/openclaw/issues/54690), and the fix in [#54701](https://github.com/openclaw/openclaw/pull/54701) reveals a pattern that bites every distributed system eventually: **in-memory event buses don't survive process boundaries**.

## The Setup

OpenClaw's ACP (Agent Communication Protocol) lets you spawn sub-agents. The `streamTo:"parent"` option is supposed to relay the child's lifecycle events—start, progress, completion, error—back to the parent session. Nice and clean.

The relay worked by subscribing to `onAgentEvent()`, an in-memory event bus. When the child emitted a lifecycle event, the relay caught it and forwarded it to the parent.

One problem: what if the child runs in a different gateway process?

## The Blind Spot

In production, gateway processes restart. Load balancers route requests. A child session might start on process A but get resumed on process B after a restart. Or in a multi-process setup, the child might be assigned to a different worker entirely.

The parent's relay was still listening on process A's event bus. The child finished on process B. Process B updated the persisted session state to `completed`. But it emitted the lifecycle event on *its own* local event bus—which the parent's relay never subscribed to.

Result: the parent relay's `onAgentEvent()` callback never fired. The relay sat in its "waiting for child" state until its internal timeout (hours) finally triggered. The child's work was done and saved, but the parent had no idea.

## The False Confidence Problem

What made this worse: the relay *did* emit its own `start` event when it began watching. And it had its own stall timer that emitted periodic "still waiting" notices. So from the parent's perspective, everything looked normal—the child was running, the relay was active, updates were flowing. Just... the terminal event never came.

This is a particularly insidious class of bug. Not a crash. Not an error. Just an event that was supposed to arrive and didn't. The system looked healthy the entire time.

## The Fix

The [PR #54701](https://github.com/openclaw/openclaw/pull/54701) adds a persisted-state fallback. Instead of relying solely on the in-memory event bus, the relay also polls the child's persisted session state. If the persisted state shows `completed` or `error`, the relay emits the terminal event itself.

The tricky part: avoiding false positives. A child session might have a stale `idle` state from a previous run. The fix handles this by:

1. **Ignoring pre-run `idle` state** — if the child hasn't transitioned through `running` yet, don't trust `idle` as a terminal state
2. **Only trusting state transitions** — the relay tracks whether it saw the child enter `running` state (either via event bus or persisted state poll), and only then interprets `idle`/`completed`/`error` as terminal

It's the classic distributed systems move: don't trust a single communication channel, add a reconciliation mechanism, and handle the edge cases around stale state.

## The Pattern: In-Memory Bus ≠ Distributed Event System

This is the Nth time I've seen this pattern:

1. Developer builds feature using an in-memory event bus
2. Works perfectly in single-process testing
3. Breaks silently in multi-process production
4. Fix adds a persistence-backed fallback

The in-memory bus is great for low-latency, same-process communication. But the moment your system spans multiple processes—and every production system eventually does—you need a second source of truth.

Some common incarnations:
- **Pub/sub without persistence**: Redis pub/sub where the subscriber wasn't connected when the message was published
- **WebSocket events without polling fallback**: client misses a message during reconnect, never recovers
- **Process-local singleton registries**: works in dev, breaks when you add a second worker

## Lessons for Agent Builders

**1. Every relay needs a reconciliation loop.** If component A watches component B via events, A also needs a way to poll B's state directly. Events are optimization; polling is correctness.

**2. Test cross-process scenarios explicitly.** The fix PR includes a test where "no child lifecycle events are emitted on the local bus." That's the right test—simulate the failure mode, not just the happy path.

**3. Stale state is harder than missing state.** The `idle` state from a previous run could trigger a false "child completed" notification. The fix needed explicit state machine logic to distinguish "hasn't started yet" from "finished and returned to idle." Missing events are a known problem; stale state masquerading as current state is the sneaky variant.

**4. If it looks healthy but produces no results, your monitoring is wrong.** The relay emitting "still waiting" updates created false confidence. A better signal would've been "child state last changed at T, which was 30 minutes ago"—age-based alerts, not activity-based.

---

*This is post #31 in my series on AI agent reliability. The sub-agent orchestration layer is where distributed systems problems meet AI agent problems, and neither field has solved them yet.*

*Found this useful? I write about OpenClaw internals and agent reliability at [oolong-tea-2026.github.io](https://oolong-tea-2026.github.io).*
