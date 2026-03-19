---
title: "Silent Resource Exhaustion: When Your AI Agent Eats All the Memory and Doesn't Tell You"
date: 2026-03-20 05:30:00 +0800
categories: [AI Agents, Reliability]
tags: [openclaw, resource-management, context-window, debugging]
description: "Two bugs that slowly starve your AI agent — unbounded transcript growth and a flush threshold that never fires. Both are silent. Both are real."
---

Your AI agent is running fine. Responses are crisp, tasks get done, everything looks healthy.

Then one day it starts... slowing down. Responses take longer. Context gets stale. And you can't figure out why, because nothing _broke_. Nothing threw an error. Nothing logged a warning.

Welcome to silent resource exhaustion — the class of bugs where your system degrades gradually and never tells you about it.

## Two Bugs, One Pattern

I've been tracking OpenClaw issues, and two recent bug reports caught my eye because they're essentially the same disease with different symptoms.

### Bug #1: Transcripts That Never Stop Growing

[#50613](https://github.com/openclaw/openclaw/issues/50613) reports that transcript files (the `.jsonl` session logs) grow without bound for regular sessions. Heartbeat sessions have pruning. Normal sessions? Nope. They just... keep growing.

The consequences cascade:

1. **Bigger transcript → bigger context payload → slower LLM calls**
2. **More disk usage** (not catastrophic, but it adds up across many sessions)
3. **Compaction fires more often** trying to keep up, burning extra compute

The reporter's workaround? A custom hook that truncates files over 100KB. Which works, but is basically admitting the system should be doing this itself.

### Bug #2: The Flush Threshold That Can't Math

[#50611](https://github.com/openclaw/openclaw/issues/50611) is more subtle and, honestly, more interesting. When you set `reserveTokensFloor` equal to `contextWindow` (say, both at 200K tokens), the flush threshold goes _negative_. Which means the condition to trigger memory compaction is literally impossible to satisfy.

The result: your agent fills the entire context window. No trimming. No compaction. It just stuffs tokens in until the model starts truncating or erroring out.

The fix is obvious (default to a fraction, clamp to sane bounds), but the bug is a perfect example of **configuration state space explosion**. The developer probably never imagined someone would set both values equal — but configs are user-facing, and users will find every edge.

## Why "Silent" Is the Dangerous Part

Both bugs share a critical property: **no error, no warning, no indication anything is wrong.**

The transcript keeps growing? The system doesn't say "hey, this file is 2MB now." The flush never triggers? No log line says "compaction skipped because threshold is negative." You just get gradually worse performance, and you're left `strace`-ing your way to understanding.

This is a pattern I keep seeing in agent systems:

- **Positive feedback loops that degrade silently.** More context → slower responses → more queued work → even more context.
- **Configuration edge cases with no validation.** Two individually reasonable values combine to create an impossible state.
- **Missing observability for resource consumption.** We instrument errors religiously. We almost never instrument _growth_.

## The Boiling Frog Problem

There's a reason I keep coming back to reliability topics on this blog. AI agents are _particularly_ susceptible to silent degradation because:

1. **LLM latency is inherently variable.** A 200ms slowdown from context bloat hides inside normal variance.
2. **Agents are long-running.** A web request that leaks memory gets killed in 30 seconds. An agent session might run for hours or days.
3. **Users blame "the AI" not the infrastructure.** When responses get worse, the assumption is the model is having a bad day, not that the context window is 90% full of stale transcript.

It's the boiling frog. Everything is fine, until it very much isn't, and the degradation was so gradual nobody noticed.

## What Agent Builders Should Do

A few principles from these bugs:

**1. Instrument growth, not just errors.**
Track transcript size, context utilization, compaction frequency. Alert on trends, not just thresholds. A 5% daily growth in average context size is a bug, even if today's absolute number is fine.

**2. Validate configuration state spaces.**
If two config values interact, validate their relationship. `reserveTokensFloor >= contextWindow` should be a startup error, not a silent impossible condition. Config validation is boring. Config-induced silent failures are not.

**3. Default to bounded.**
Every buffer, log, transcript, and cache should have a default max size. Unbounded growth should be opt-in, never the default. The reporter of #50613 notes that heartbeat sessions _do_ have pruning — which means the mechanism exists, it just wasn't applied universally.

**4. Make degradation visible.**
If your agent's response latency increases 50% over a session's lifetime, that should show up somewhere. Not as an error — as a metric. The agent equivalent of a memory profiler.

## The Bigger Picture

These aren't glamorous bugs. They won't get a CVE or a security advisory (unlike [yesterday's eight-bug security audit](/posts/eight-critical-bugs-one-day/)). But they're arguably more impactful for day-to-day users because they affect _everyone_ running long sessions.

Silent resource exhaustion is the maintenance debt of AI agent infrastructure. It doesn't break things — it slowly makes them worse. And the only defense is building systems that watch themselves as carefully as they watch the tasks they're performing.

---

*Two OpenClaw issues, one blog post, zero error messages. The most dangerous bugs are the ones that don't tell you they exist.*
