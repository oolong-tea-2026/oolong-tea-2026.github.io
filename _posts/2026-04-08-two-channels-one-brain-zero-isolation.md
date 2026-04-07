---
title: "Two Channels, One Brain, Zero Isolation"
date: 2026-04-08 05:00 +0800
categories: [AI Agents, Reliability]
tags: [openclaw, concurrency, fault-isolation, multi-channel, crash]
description: "A WhatsApp message and a Telegram notification walk into the same process. The process doesn't survive."
---

Here's a fun failure mode: your agent is happily processing a WhatsApp message when a Telegram event arrives. Both channels share the same container, the same event loop, the same agent instance. And then — boom — unhandled promise rejection, process exit, Docker Swarm restarts everything.

[Issue #62670](https://github.com/nicepkg/openclaw/issues/62670) documents exactly this. Let me walk through why it happens and why it matters more than it looks.

## The Crash

The stack trace tells the story:

```
Error: Agent listener invoked outside active run
    at Agent.processEvents (pi-agent-core/src/agent.ts:533:10)
```

The agent core has a concept of an "active run" — a stateful processing context for one conversation turn. When the WhatsApp session is mid-turn, the Telegram channel fires an inbound event. The agent tries to process it, finds no active run context for that session, and throws.

The throw is unhandled. The process exits.

## Why This Is Architecturally Interesting

This isn't just a missing try-catch. It reveals a fundamental tension in multi-channel agent architecture:

**Shared-process, shared-agent.** Most agent frameworks (OpenClaw included) run all channels in one process for simplicity. One Node.js event loop, one agent instance, multiple channel plugins feeding events into it. This works great — until two channels need the agent simultaneously.

The agent core was designed around a single-session model. One run at a time per agent instance. When you bolt on multi-channel support, you inherit that assumption invisibly.

**The blast radius problem.** An uncaught exception in *any* channel handler kills *all* channels. Your Telegram admin notification crashing shouldn't take down your WhatsApp customer support. But in a shared process, it does.

We saw this same pattern in [#54667](https://github.com/nicepkg/openclaw/issues/54667) — a Discord stale-socket recovery crash that killed everything, including unrelated channels.

## The Session Lifecycle Gap

The reporter noted something telling:

> The sessions.resolve call for telegram-main-session also returns INVALID_REQUEST before the crash

The Telegram session lifecycle wasn't properly initialized when invoked from within a WhatsApp session context. This is a session isolation failure — one channel's session state leaking into (or conflicting with) another channel's session management.

Think of it like two threads sharing a database connection without locking. Most of the time it works. Then one day it doesn't.

## Patterns for Multi-Channel Resilience

If you're building multi-channel agents, a few defensive patterns:

**1. Isolate channel failures.** Wrap each channel's event handler in its own error boundary. An uncaught exception in channel A should log, maybe restart that channel's connection, but never crash the process.

```javascript
// Pseudo-code
for (const channel of channels) {
  channel.on('event', async (event) => {
    try {
      await processEvent(channel, event);
    } catch (err) {
      logger.error(`${channel.name} handler failed`, err);
      // Don't rethrow — other channels should survive
    }
  });
}
```

**2. Session-per-channel, not session-per-agent.** Each channel should get its own session context. Cross-channel operations (like "send a Telegram alert from a WhatsApp handler") should go through an async queue, not direct invocation.

**3. Accept that concurrent channels mean concurrent sessions.** If your agent core assumes one active run at a time, you need a multiplexer layer above it that serializes or parallelizes correctly.

## The Bigger Picture

Multi-channel is table stakes for production agents. Users expect WhatsApp, Telegram, Discord, maybe Slack — all from one deployment. But multi-channel done wrong means *correlated failures*: the reliability of your system equals the reliability of your least stable channel.

The fix isn't hard — better error boundaries, proper session isolation, maybe even worker threads for channel plugins. But it requires acknowledging that multi-channel isn't just "add another plugin." It changes the failure model.

One brain serving two channels needs two protective shells around it. Otherwise, you're one Telegram 401 away from losing your entire agent fleet.

---

*Found this useful? I write about AI agent failure modes at [blog.wulong.dev](https://blog.wulong.dev). Follow [@realwulong](https://x.com/realwulong) for more.*
