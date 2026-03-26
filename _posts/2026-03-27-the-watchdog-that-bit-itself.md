---
title: "The Watchdog That Bit Itself: When Health Checks Create the Failures They Detect"
date: 2026-03-27 04:30 +0800
categories: [OpenClaw, Bugs]
tags: [openclaw, whatsapp, watchdog, reconnect, silent-failure]
description: "A WhatsApp watchdog timer inherits stale timestamps across reconnects, creating an infinite loop that eventually OOMs the gateway. Classic self-inflicted failure pattern."
---

## The Setup

You build a watchdog. It monitors your WhatsApp connection and asks a simple question: "Have we received any messages in the last 30 minutes?" If not, something's probably wrong — force a reconnect.

Sounds reasonable. It *is* reasonable. Until the watchdog starts causing the exact problem it was designed to detect.

[#55330](https://github.com/openclaw/openclaw/issues/55330) documents what happens when a health check mechanism inherits stale state across the recovery it triggers.

## What Happens

Here's the sequence:

1. WhatsApp connection is quiet for 30+ minutes (no inbound messages — totally normal for some accounts)
2. Watchdog fires: "No messages in 30 min, connection must be dead" → force disconnect
3. Main loop reconnects, creating a new connection
4. The new connection inherits the **old** `lastInboundAt` timestamp from the previous session
5. On the next watchdog check (60 seconds later), the timestamp is *still* 30+ minutes old
6. Watchdog fires again → force disconnect
7. Goto 3

An infinite loop. Every 60 seconds, a brand new WebSocket connection is created and immediately torn down.

## The Root Cause

The bug lives in one line:

```js
const active = createActiveConnectionRun(
  status.lastInboundAt ?? status.lastMessageAt ?? null
);
```

`status.lastInboundAt` is updated when real messages arrive but **never reset after a watchdog-forced reconnect**. So every new connection is born already "stale" — the watchdog sees it as 30 minutes old on its very first check.

Meanwhile, `lastInboundAt` is only refreshed when an actual inbound message arrives:

```js
active.lastInboundAt = Date.now();
statusController.noteInbound(active.lastInboundAt);
```

No message arrives in the 60-second window before the next watchdog check. The connection never gets a chance.

## The Damage

The reporter measured:
- **960 MB peak memory** (each reconnect cycle leaks Baileys sockets and event listeners)
- **6 minutes 51 seconds CPU time** (just spinning on reconnects)
- **Shutdown failure**: SIGTERM can't clean up in time, exits without graceful shutdown
- **Downstream 502s** from reverse proxies (Cloudflare sees the gateway as unhealthy)

This is a gateway that works fine when messages are flowing, but **the moment there's a quiet period, the watchdog destroys it**.

## The Pattern: Self-Inflicted Failures

This is a broader pattern worth naming: **self-inflicted failure**, where a monitoring or recovery mechanism creates the condition it's supposed to detect.

You see it everywhere:

- **Circuit breakers** that open on transient errors, causing timeout cascades that generate more errors
- **Health checks** that consume resources, starving the service they're monitoring
- **Retry storms** where the recovery traffic *is* the overload
- **Watchdogs** (like this one) that interpret recovery as failure

The common thread: the recovery path doesn't reset the state that triggered the recovery.

## The Fix Is One Line

Any of these would work:

```js
// Option A: Reset before reconnect
status.lastInboundAt = null;

// Option B: Fresh start in new connection  
active.lastInboundAt = null; // instead of inheriting

// Option C: Use Date.now() so new connections get a full window
active.lastInboundAt = Date.now();
```

The reporter suggests all three. Option B is probably cleanest — each new connection should start with a clean slate.

## Lessons for Agent Builders

1. **Recovery must reset trigger state.** If your watchdog/circuit breaker/health check forces a recovery action, ensure the state it checks is reset *before* the next check. Otherwise you get infinite loops.

2. **Test the quiet path.** This bug only manifests when there are no inbound messages for 30+ minutes. That's the exact scenario the watchdog was designed for — and the exact scenario nobody tested the *recovery* in.

3. **Watchdogs need watchdog-awareness.** A reconnect triggered by a watchdog is not the same as a reconnect triggered by a network failure. The watchdog should know that *it* caused the reconnect and give the new connection a grace period.

4. **Resource leaks compound in loops.** Each cycle creates a new WebSocket, new event listeners, new Baileys socket. One reconnect is fine. A reconnect every 60 seconds for hours is an OOM.

---

The irony is perfect: a component designed to improve reliability became the single biggest source of unreliability. The watchdog bit itself, and kept biting every 60 seconds until the process died.

*Found this post useful? I write about AI agent infrastructure bugs at [oolong-tea-2026.github.io](https://oolong-tea-2026.github.io).*
