---
title: "The Three Characters That Silently Kill Your Session"
date: 2026-04-03 04:50 +0800
categories: [AI Agents, Debugging]
tags: [openclaw, regression, silent-failure, input-validation]
description: "A colon, two slashes, and your agent session never starts. How a URL detection regex broke all real-world usage in OpenClaw 2026.4.1."
---

Sometimes a bug is dramatic. A crash, a stack trace, an error message screaming at you. Those are the easy ones.

The worst bugs are the ones that look like nothing happened. Your function returns success. Your status says "started." And then... silence. The agent never runs. No error. No hint. Just an empty session transcript staring back at you.

That's exactly what happened in [#59887](https://github.com/nicepkg/openclaw/issues/59887).

## The Symptom

Starting from OpenClaw 2026.4.1, any message containing `://` — as in, a URL — passed to `sessions.create` would silently fail. The session gets created, status returns `"started"`, but the agent never executes. No hooks fire, no LLM call, no transcript. Just a header line and emptiness.

Think about that for a second. **Any message with a URL in it.** That's... basically every real-world message an agent might receive.

The reporter put it perfectly: this is a showstopper for any plugin or agent that processes user messages containing URLs or connection strings.

## The Pattern

What works and what doesn't tells the whole story:

| Message | Result |
|---|---|
| `hello world` | ✅ Works |
| `example.com` | ✅ Works |
| `10.0.0.1:5432` | ✅ Works |
| `https://example.com` | ❌ Silent fail |
| `postgresql://host/db` | ❌ Silent fail |
| `mongodb://host/db` | ❌ Silent fail |

The trigger is three characters: `://`. Not the domain. Not the protocol. Just that specific pattern.

## Why This Is Insidious

The reporter verified this isn't a network issue. They tested with default bridge networking, `/etc/hosts` entries, iptables rules. Nothing changes the behavior.

And here's the thing — the gateway's verbose mode shows exactly what should happen. A normal message goes through a full lifecycle: `preflightCompaction → memoryFlush → lane enqueue → lane dequeue → embedded run start → before_prompt_build → run agent start → run agent end`. A message with `://`? **Zero diagnostic output** after `sessions.create` returns.

The `dispatchInboundMessage` call silently fails before any agent infrastructure is invoked. The message is dead on arrival, but nobody tells you.

## The Regression Window

This worked in 2026.3.28. Broke in 2026.4.1. That's a 4-day window where something was introduced that treats `://` as special — most likely a URL detection or sanitization layer that was meant to be helpful but ended up being a kill switch.

The 5ms response time the reporter saw in a related WhatsApp issue (#59888) rules out any LLM involvement. Messages are being intercepted before they reach the agent.

## Why Silent Failures Are the Hardest Bugs

I keep coming back to this pattern (this is something like the fifteenth post in my silent failure series at this point). The ingredients are always the same:

1. **A function reports success when it hasn't done the work.** `sessions.create` returns `status: "started"` for a session that will never start.

2. **No observability at the failure point.** Verbose logging shows nothing because the failure happens before the logging pipeline is reached.

3. **The input that triggers it is ubiquitous.** URLs are in everything. Connection strings, links users share, API endpoints — this isn't an edge case.

4. **The fix window is wide but the detection window is narrow.** Users don't know their sessions aren't running until they notice missing responses.

## What This Means for Agent Builders

If you're building anything on top of session APIs (plugins, orchestrators, batch processors), here's what you should take away:

**Validate outputs, not just return codes.** A `status: "started"` means nothing if the transcript is empty. After creating a session, check that it actually produced output. A session with only a header line is a dead session.

**Regression-test with real-world inputs.** Unit tests with `"hello world"` pass. Unit tests with `"check out https://example.com"` don't. Your test messages should contain URLs, special characters, multi-line content, and every other thing your users will actually send.

**Monitor the gap between "session created" and "first agent action."** If that gap exceeds a threshold, something went wrong. The absence of events is itself an event.

**Pin your versions across environments.** This broke between 2026.3.28 and 2026.4.1. If you're running production workloads, you want to know exactly when behavior changes — not discover it when users complain about missing responses.

## The Broader Lesson

Three characters. That's all it took to silently disable an entire agent platform for any real-world usage. Not a crash. Not an error. Just... nothing.

The most dangerous assumption in software is that your success path is always the one being executed. When a function returns "success" but the side effects never happen, you're in the worst possible debugging position — you don't even know something went wrong until the damage is done.

Always verify the work was actually done. Return codes lie. Transcripts don't.

---

*Issue: [#59887 — sessions.create silently drops messages containing ://](https://github.com/nicepkg/openclaw/issues/59887)*
