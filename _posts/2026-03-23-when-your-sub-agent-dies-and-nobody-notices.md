---
title: "When Your Sub-Agent Dies and Nobody Notices"
date: 2026-03-23 04:30:00 +0800
categories: [OpenClaw, Analysis]
tags: [openclaw, acp, subagents, silent-failure, observability]
description: "ACP sessions can die without emitting any event back to the parent. Your orchestrator keeps waiting. Your user keeps hoping. Nobody knows the task is dead."
---

Another day, another silent failure mode. This time it's not about vision or config or fallback chains — it's about the fundamental contract between a parent agent and the sub-agents it spawns.

[Issue #52452](https://github.com/nicepkg/openclaw/issues/52452) documents a pattern that should terrify anyone building multi-agent systems: **sub-agent processes die without telling anyone**.

## The Setup

OpenClaw supports spawning isolated sessions via ACP (Agent Communication Protocol). You call `sessions_spawn(runtime='acp')`, hand off a task, and wait for a completion event. Simple enough in theory.

In practice, the reporter ran 3 ACP sessions. All 3 failed. Zero completion events were emitted.

- **Run 1:** `Permission denied` error. Logged to gateway logs. Parent never notified. Parent told the user "it's running" for **40 minutes** while the task had died instantly.
- **Run 2:** Process started, did partial work, got killed by a gateway restart. No error event.
- **Run 3:** Process started, worked on 8 files, died silently. `ps aux` confirmed it was gone. Parent: still waiting.

## The Fire-and-Forget Problem

This is the classic distributed systems mistake: **fire and forget without a dead letter queue**.

The parent spawns a child process and assumes it will eventually call back. But what if:

- The child crashes before it can emit anything?
- The child is killed externally (OOM, restart, signal)?
- The child hits a permission error on startup?
- The runtime itself fails to start the child?

In all these cases, the parent is left in limbo. It can't check status (`sessions_history` returns `forbidden`). It can't list active sub-agents (`subagents list` doesn't show ACP sessions). The only way to discover the failure is manually grepping gateway logs.

That's not observability. That's archeology.

## Why This Matters More Than You Think

Multi-agent orchestration is the hot new thing. Everyone's building agent swarms, task decomposition pipelines, and coding assistants that spawn sub-agents for parallel work. But the reliability story is still stuck in single-agent thinking.

Consider the failure modes:

```
Parent Agent                    ACP Session
    |                               |
    |--- spawn task --------------->|
    |                               | (Permission denied!)
    |    (waiting...)               | (dead)
    |    (still waiting...)         |
    |    "It's running!"            |
    |    (40 minutes later...)      |
    |    ???                        |
```

The parent has no way to distinguish between "child is working" and "child died on startup." Without health checks or completion guarantees, every spawned session is Schrödinger's agent.

## What's Missing

The issue identifies four gaps:

1. **No failure callback.** When an ACP session dies, the parent must be notified. Period. This isn't optional — it's the minimum viable contract for task delegation.

2. **No progress visibility.** You can't query ACP session status from the parent. `subagents list` is blind to ACP sessions. This makes debugging impossible without log access.

3. **No process exit monitoring.** If the child process dies unexpectedly, the runtime should detect `SIGCHLD` (or equivalent) and synthesize an error event. This is process management 101.

4. **Silent startup failures.** Permission errors, missing binaries, config issues — all of these should produce immediate error events, not silent log entries.

## The Pattern: Supervision Trees

Erlang solved this decades ago with supervision trees. The principle is simple: **every child process has a supervisor that monitors it and takes action on failure**.

For agent systems, the minimum is:

- **Heartbeat or watchdog:** Parent periodically checks if child is alive
- **Exit trapping:** Runtime detects child death and notifies parent
- **Startup confirmation:** Child must confirm successful startup before parent considers it "running"
- **Timeout with escalation:** If no completion event arrives within a deadline, parent treats it as failure

None of these are exotic. They're table stakes for any system that delegates work to another process.

## Lessons for Agent Builders

1. **Never trust fire-and-forget.** If you spawn a sub-agent, you need a completion guarantee. Set timeouts. Monitor exits. Treat silence as failure after a deadline.

2. **Startup is the most dangerous phase.** Permission errors, missing deps, config issues — most failures happen in the first second. Require a "ready" signal before reporting success to the user.

3. **Observability across process boundaries.** If your parent can't query child status, you're flying blind. Logs aren't observability — they're forensics.

4. **Design for the unhappy path.** The happy path is easy: child runs, completes, reports back. The unhappy path — crashes, kills, permission denials, partial work — is where reliability lives.

## The Bigger Picture

This is the sixth post in what's becoming an accidental "silent failure" series. The common thread? **Agent systems fail quietly by default.** Config changes don't apply (#51251). Fallback chains don't cascade (#51209). Vision gets disabled without warning (#51869). MIME sniffing blocks image processing (#51881). Success responses contain no data (#51857). And now, sub-agents die without notification (#52452).

Each individual bug is fixable. But the pattern suggests something deeper: agent frameworks are still optimized for the happy path. The error handling, observability, and failure recovery that we take for granted in web services and distributed systems haven't fully arrived in the agent world yet.

Until they do, assume your sub-agents are lying to you. Or worse — assume they're already dead and you just don't know it yet.

---

*Wu Long is an independent developer and OpenClaw contributor who writes about AI agent architecture at unreasonable hours. His sub-agents, at least, are honest about being unreliable.*
