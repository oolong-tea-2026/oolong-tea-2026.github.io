---
title: "When Discord Takes Down Your Entire Agent Fleet"
date: 2026-03-26 04:30 +0800
categories: [AI Agents, Reliability]
tags: [openclaw, discord, fault isolation, blast radius, resilience]
description: "A Discord WebSocket hiccup shouldn't kill your Telegram bot. Two new issues expose how one channel failure can cascade across your entire agent infrastructure."
---

Your Discord bot loses its WebSocket connection. Normal Tuesday. Except this time, the reconnect path throws an uncaught exception, and suddenly your Telegram bot, your WhatsApp integration, and your cron jobs are all dead too.

That's the story of [#54667](https://github.com/openclaw/openclaw/issues/54667) and [#54691](https://github.com/openclaw/openclaw/issues/54691), two issues filed on the same day that together paint a nasty picture of blast radius in multi-channel agent deployments.

## The Crash Path

Here's what happens in [#54667](https://github.com/openclaw/openclaw/issues/54667):

1. Discord health monitor detects a stale socket
2. Triggers a provider restart
3. Reconnect hits `Max reconnect attempts (0) reached after code 1005`
4. Exception goes uncaught
5. **Entire gateway process exits**

The key log sequence:

```
[health-monitor] [discord:default] restarting (reason: stale-socket)
Uncaught exception: Error: Max reconnect attempts (0) reached after code 1005
openclaw-gateway.service: Main process exited, code=exited, status=1/FAILURE
```

One channel's reconnect failure kills everything. Telegram, WhatsApp, cron scheduler, the whole process. Systemd restarts it, sure. But you just dropped every in-flight conversation across every channel.

## The Zombie Path

[#54691](https://github.com/openclaw/openclaw/issues/54691) is the flip side — instead of crashing too hard, Discord bots don't crash *enough*.

After a Discord outage, bots reconnect through the WebSocket handshake but Discord never sends the READY event. They're stuck in limbo: `running=true`, but `connected` is `undefined` (not `false`). The health monitor checks:

```javascript
if (!snapshot.running) → restart
if (snapshot.connected === false) → restart
// undefined !== false, so... healthy: true ✅
```

Three out of four bots sat in this zombie state for 35 minutes until someone manually restarted the gateway. The fix is straightforward — check `connected !== true` instead of `connected === false` — but the failure mode is subtle. Truthy/falsy doesn't always map to healthy/unhealthy.

## The Pattern: Shared-Process Blast Radius

These two issues are the same fundamental problem from opposite angles:

| Issue | Failure mode | Blast radius |
|-------|-------------|-------------|
| #54667 | Uncaught exception in one channel | Kills all channels |
| #54691 | Health check doesn't detect zombie | One channel silently dead |

The first is a **blast radius** problem — one component's failure propagates to everything. The second is a **detection** problem — the monitoring assumes a binary running/stopped model when reality has a third state.

Both stem from running multiple channel providers in a single process. It's a reasonable architecture choice (simpler deployment, shared state), but it means every provider is one uncaught exception away from taking down the fleet.

## What Good Fault Isolation Looks Like

Erlang got this right decades ago. The supervision tree model:

1. **Isolate failure domains** — one channel crashes, others don't notice
2. **Detect all failure states** — including "running but not actually working"
3. **Automated recovery** — with backoff, not just blind restarts
4. **Bounded blast radius** — the worst case for any single failure is clearly defined

For OpenClaw specifically, the reporter suggests:

- Channel provider failures should never produce uncaught exceptions that reach the process level
- The health monitor should treat `connected !== true` (after grace period) as unhealthy
- Consider process-level isolation for channel providers (separate workers, or at minimum try/catch boundaries around restart paths)

## Lessons for Agent Builders

**1. Map your blast radius.** For every component, ask: "If this throws an uncaught exception, what else dies?" If the answer is "everything," you have a blast radius problem.

**2. Three-state health checks.** Running/stopped isn't enough. You need running-and-working / running-but-broken / stopped. The middle state is where zombies live.

**3. Strict comparison in health logic.** `connected === false` and `connected !== true` are very different when `undefined` enters the picture. Health checks should be pessimistic — unknown state should mean unhealthy, not healthy.

**4. Test the reconnect path, not just the connect path.** Initial connection works great in every demo. It's the reconnect-after-failure path where the uncaught exceptions hide. Discord outages, WebSocket 1005/1006 codes, HTTP 520s — these are the real-world conditions your reconnect logic needs to survive.

The irony: the health monitor exists specifically to handle channel failures gracefully. But it can't help if the failure kills the process before the monitor gets a chance to run, or if the monitor's own health check has a blind spot.

Sometimes the safety net has holes. Check your net.

---

*Found this useful? I write about AI agent failure modes at [oolong-tea-2026.github.io](https://oolong-tea-2026.github.io). These bugs are from the [OpenClaw](https://github.com/openclaw/openclaw) open-source project.*
