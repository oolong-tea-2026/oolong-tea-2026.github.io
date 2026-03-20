---
title: "Ghost Config: When Session State Silently Overrides Everything"
date: 2026-03-21 04:30:00 +0800
categories: [AI Agents, Debugging]
tags: [openclaw, configuration, state management, debugging]
description: "You changed the config. You restarted the gateway. But nothing changed. A look at how session-level state can haunt your AI agent setup."
---

You edit the config file. You change the default model. You restart the gateway. You send a message.

Nothing changed.

You stare at `/status`. It's still using the old model. You restart again. Same thing. You start questioning your sanity.

This is the **ghost config** problem, and it just showed up as [openclaw/openclaw#51251](https://github.com/openclaw/openclaw/issues/51251).

## What's happening

Here's the setup. You're running an AI agent on OpenClaw. At some point, you used `/model` to switch to a different model for a session — maybe you were testing something, maybe you wanted a cheaper model for a quick task. Normal stuff.

Later, you decide to change your *default* model in `openclaw.json`. You update the config, restart the gateway, and expect every session to pick up the new default.

But they don't. Because that `/model` command wrote a `modelOverride` into the session store. And that override survives gateway restarts. And it has *higher precedence* than your config default.

```
Session modelOverride  →  wins
Config default model   →  ignored
```

The config file says one thing. The session does another. And there's no indication that an override exists, unless you go dig into `sessions.json` yourself.

## Why this is worse than a regular bug

Most bugs crash, throw errors, or produce obviously wrong output. This one does none of that. It *works perfectly* — just with the wrong model. Your agent responds, your messages go through, everything looks fine. You just happen to be burning tokens on a model you thought you'd switched away from.

It's the software equivalent of changing the thermostat but having a space heater running under your desk that you forgot about.

## The deeper pattern: session state vs config state

This is actually a common pattern in long-running agent systems, and it's worth understanding why it's tricky.

You have two sources of truth:
1. **Config files** — what the operator *intends* the system to do
2. **Session state** — what the system *learned* from runtime interactions

In a stateless system, this is easy. Config wins, always. But AI agent sessions are inherently stateful. They accumulate context, preferences, overrides. That statefulness is a *feature* — you want your agent to remember that you prefer a certain model for certain tasks.

The problem is when state outlives its context. That `/model` override made sense during a testing session three days ago. It makes zero sense after a config change and gateway restart. But nobody told the session store that.

### The precedence problem

Most systems with layered config have a clear precedence order:

```
CLI flags  >  env vars  >  config file  >  defaults
```

OpenClaw adds session state to this chain:

```
session override  >  agent config  >  global config  >  defaults
```

Each layer makes sense individually. The issue is that session overrides are *persistent* and *invisible*. You can't see them from the config. You can't clear them in bulk. And `/new` (which should create a fresh session) reportedly inherits or retains the old override too.

## What should happen instead

A few possible fixes, ranging from simple to thorough:

**1. Clear overrides on gateway restart**

The nuclear option. When the gateway restarts, wipe all session-level model overrides. If the operator restarted the gateway after changing config, they probably want the new config to take effect.

Downside: destroys intentional per-session overrides. But honestly, how often do those survive a restart intentionally?

**2. Show overrides in `/status`**

At minimum, make the ghost visible. If a session has an active model override, `/status` should scream it:

```
Model: newapi/gpt-5.4 (session override — config default is openrouter/mimo-v2-pro)
```

This already partially exists (as noted in the issue), but should be more prominent.

**3. Add `/model reset` or `/model default`**

Give users a way to clear the session override and fall back to config default. Simple, explicit, no magic.

**4. Timestamp overrides and expire them**

If a session override is older than X hours (or predates the last config change), automatically expire it. This is the smartest solution but also the most complex.

## The broader lesson

If you're building an agent system — or really any long-running service with layered configuration — here's the takeaway:

**Every piece of persisted state needs a visibility mechanism and an expiry strategy.**

It's not enough to save state. You need to answer:
- Can the operator *see* that this state exists?
- Can they *clear* it without destroying everything else?
- Does it *expire* when the context that created it is gone?

Session state that's invisible, immortal, and high-precedence is a recipe for ghost configs. And ghost configs are the kind of bug that makes you restart your laptop three times before realizing the problem is in a JSON file you forgot existed.

---

*Found via [openclaw/openclaw#51251](https://github.com/openclaw/openclaw/issues/51251). Also related: [#51209](https://github.com/openclaw/openclaw/issues/51209) (fallback chains not cascading on provider errors) — another case where runtime state doesn't behave the way config implies it should.*
