---
title: "The Tool Parameter Your LLM Should Never See"
description: "How exposing an internal runtime enum to the model creates a self-reinforcing spawn failure that's nearly impossible to debug."
date: 2026-04-10 05:00 +0800
categories: [AI Agents, Tool Design]
tags: [openclaw, tool-design, llm, api-surface, footgun]
---

There's a class of bug that only exists because we forgot who's calling our APIs.

When a human developer calls `sessions_spawn`, they read the docs. They know `runtime: "subagent"` means "delegate to another in-config agent" and `runtime: "acp"` means "spawn an external binary via the Agent Communication Protocol." They pick the right one.

An LLM doesn't read docs. It reads the tool schema, maybe the parameter description, and then it *guesses*.

## The Bug

[OpenClaw #63914](https://github.com/nicepkg/openclaw/issues/63914) describes a deployment with a router agent (Claude Haiku 4.5) that delegates work to specialist agents configured in the same gateway. The intended call:

```json
{
  "name": "sessions_spawn",
  "runtime": "subagent",
  "agentId": "pleres",
  "task": "Draft the quarterly report"
}
```

What the model sometimes emits instead:

```json
{
  "name": "sessions_spawn",
  "runtime": "acp",
  "agentId": "pleres",
  "task": "Draft the quarterly report"
}
```

The difference is one string. The effect is total: the system tries to `child_process.spawn("pleres")` as a literal binary on `$PATH`, fails with `spawn_failed`, and the user gets nothing.

## Why This Keeps Happening

Here's the insidious part. The failure lands in the conversation history:

```
errorCode: spawn_failed
error: Failed to spawn agent command: pleres
```

The model sees this, retries... and often picks `runtime: "acp"` again. Why? Because:

1. The error doesn't say "wrong runtime." It says "spawn failed."
2. The model's instinct on failure is to retry with minor tweaks — maybe a different task phrasing, not a different runtime value.
3. Once `runtime: "acp"` appears in context, it has a recency anchor that biases the next attempt.

Five failures in six hours, same session. The model learned the wrong thing from its own mistakes.

## The Design Problem

This isn't really about one enum. It's about **who your tool's audience is**.

Traditional API design assumes the caller understands the semantic difference between options. LLM tool design can't assume that. The model picks from a schema based on statistical patterns, not understanding.

The `runtime` parameter is a plumbing detail. It controls *how* the spawn happens at the infrastructure level — in-process delegation vs. external binary protocol. From the model's perspective, there's no meaningful distinction. Both achieve "run this task on another agent." The model shouldn't need to know (or care) about the process boundary.

## What Good LLM Tool Design Looks Like

A few principles I keep seeing reinforced:

**1. Don't expose implementation details as parameters.**

If two code paths produce the same *logical* outcome (task → agent → result), the tool should pick the right path internally. The model says *what* it wants; the system figures out *how*.

For `sessions_spawn`, the fix is straightforward: if `agentId` matches an in-config agent, use subagent mode. If it matches a known ACP binary, use ACP mode. The model never sees the enum.

**2. Error messages should guide the model toward recovery.**

"Failed to spawn agent command: pleres" is accurate for a developer reading logs. For a model, it's useless. A better error:

> Agent "pleres" is an in-config agent. Remove the `runtime` parameter and retry — routing is automatic.

Now the model has a corrective signal instead of a dead end.

**3. Schema surface area = hallucination surface area.**

Every optional parameter is a chance for the model to fill in something plausible but wrong. Enums are especially dangerous because they give the model a small, confident set of options — both of which *look* correct.

The rule: if a parameter isn't meaningful to the model's decision-making, don't put it in the schema.

**4. Test with dumb models, not smart ones.**

GPT-5.4 might pick the right runtime every time. Claude Haiku 4.5 doesn't. If your tool works with the smartest model but breaks with the cheapest one people actually deploy, your tool has a bug.

## The Broader Pattern

This connects to something I've been noticing across agent frameworks: the tool surface is the new API surface, and we haven't fully internalized what that means.

Traditional APIs are called by code. The caller is precise, deterministic, and doesn't hallucinate. LLM tool APIs are called by a probabilistic system that will absolutely try every valid (and some invalid) combination of your parameters if given enough turns.

Design accordingly:

- Minimize parameters
- Make invalid states unrepresentable in the schema
- Auto-detect what you can
- Write errors that teach, not just report

The reporter in #63914 has 13 agents in their deployment. That's a serious production setup, not a toy. And it was brought down by a two-value enum that should have been invisible.

---

*Every parameter you expose to a model is a question you're asking it to answer. Make sure it's a question worth asking.*
