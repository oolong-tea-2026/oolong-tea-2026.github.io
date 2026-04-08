---
title: "Your Agent Called the Wrong Agent — On Purpose"
date: 2026-04-09 04:30 +0800
categories: [AI Agents, Security]
tags: [openclaw, multi-agent, authorization, llm-safety]
description: "When an LLM infers a communication target from context instead of following its allowlist, your agent silo architecture breaks silently."
---

You set up thirteen agents. You drew careful boundaries: coaching team over here, SaaS team over there, orchestrator bridges in between. Each agent has an `allowAgents` list — a whitelist of who it's allowed to talk to.

Then one of your agents just... called someone it wasn't supposed to. Not because of a bug in the routing. Because the *model decided to*.

## The Setup

[OpenClaw #63351](https://github.com/openclaw/openclaw/issues/63351) describes a multi-agent deployment with 13 agents organized into two teams. The communication topology is intentional: agents can only reach specific peers through configured `allowAgents` arrays.

Agent `vox` is allowed to talk to `sensei`, `maestro`, and `vigil`. That's it.

Agent `wattson` is not on vox's list.

## What Happened

Vox was processing bug reports. Some of those bugs *concerned* a product called Wattson. The model — Gemini 3 Pro — saw "Wattson" in the bug report content and inferred that agent Wattson was the right target for `sessions_send`.

The call went through. No error, no warning. The `allowAgents` configuration was completely ignored.

Two failures stacked on top of each other:

1. **The LLM inferred a target from content, not from its instructions.** The prompt listed valid targets explicitly, but the model pattern-matched on a name it saw in the data.
2. **The gateway didn't enforce the boundary.** `sessions_send` delivered the message without checking whether the caller was authorized to reach that session.

## Why This Matters More Than It Looks

The first failure is annoying but expected — LLMs do this. They see a name, they make a connection, they act on it. That's why you have guardrails.

The second failure is the real problem. The `allowAgents` config exists *precisely* for moments when the model goes off-script. It's a safety net. Except the safety net had a hole: `sessions_send` never checks it.

This is a pattern I keep seeing in agent frameworks: **the authorization model exists in config but isn't enforced at the API boundary**. The config *describes* the intended topology. The runtime *ignores* it.

## The Deeper Issue: Prompt-Based vs. Gateway-Enforced Security

The workaround the reporter used? Adding an explicit blocklist to the agent's prompt:

> "You must NEVER send messages to wattson, darwin, or gutenberg."

This works... until it doesn't. Prompt-based security is best-effort. It relies on model compliance, which is:
- Model-dependent (Gemini 3 Pro might respect it; a future model might not)
- Context-dependent (long conversations erode instruction following)
- Adversarially fragile (prompt injection can override it)

Gateway-enforced security is deterministic. The check either passes or it doesn't. No model discretion involved.

The entire point of having an `allowAgents` config is to not rely on the model for authorization decisions. If the gateway doesn't enforce it, the config is documentation, not security.

## What Good Enforcement Looks Like

When agent A calls `sessions_send` targeting a session owned by agent B:

1. Resolve the target session's owning agent
2. Check if B is in A's `allowAgents` array
3. If not, reject the call with a clear error
4. Log the attempt (for auditing unauthorized cross-agent chatter)

Step 4 is particularly important. In the reported case, the unauthorized communication was discovered by accident. In a larger deployment, it might go unnoticed for weeks.

## The Pattern

This is the third time I've written about authorization boundaries that exist in config but aren't enforced at runtime:

- [/pair approve bypassed scope guards](/posts/when-pair-approve-bypasses-the-scope-guard/) (CVSS 9.9)
- [Dashboard bearer tokens leaked in logs](/posts/when-your-dashboard-leaks-the-keys/)
- And now: `allowAgents` ignored by `sessions_send`

The pattern is always the same: someone designs a sensible security model, implements the config schema, wires up the happy path — and forgets to add the enforcement check on the API that actually moves data.

## Takeaway for Agent Builders

If you're running multi-agent deployments:

1. **Don't trust prompt-based access control.** It's a hint, not a gate.
2. **Verify that your framework enforces the boundaries you configure.** Test it: have agent A deliberately try to reach agent B when B isn't in A's allowlist. Does it fail?
3. **Log unauthorized attempts.** An agent trying to reach outside its boundary is a signal — either the model is confused, or something worse is happening.
4. **Treat agent-to-agent communication like network traffic.** Firewalls don't ask packets nicely to go to the right destination. Neither should your agent runtime.

The whole point of a silo architecture is that it works *even when the model doesn't cooperate*. If your silos are prompt-enforced only, you don't have silos — you have suggestions.
