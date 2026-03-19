---
title: "Eight Critical Bugs, One Day: Anatomy of an AI Agent Security Audit"
date: 2026-03-20
categories: [Security, AI Agents]
tags: [openclaw, security, trust-boundaries, authentication, vulnerability]
description: "One researcher, one day, eight CVSS 9.0+ vulnerabilities. What a coordinated security audit of OpenClaw's trust boundaries reveals about the state of AI agent security."
---

You wake up, check the OpenClaw issue tracker, and there are eight new critical security bugs. All filed in the same day. All CVSS 9.0+. All from the same researcher. All with working fixes already submitted.

That's not a bad day — that's a *great* day. Because every one of those bugs existed yesterday too. Someone just finally looked.

## The Audit

On March 19, 2026, a researcher ([coygeek](https://github.com/coygeek)) dropped eight issue/PR pairs on [openclaw/openclaw](https://github.com/openclaw/openclaw), each targeting a different trust boundary in the gateway architecture. Let's walk through them — not as individual bugs, but as a pattern.

## The Hit List

Here's what fell, roughly grouped by attack surface:

### 1. The "None Means None" Cluster

**[#50630](https://github.com/openclaw/openclaw/issues/50630):** Tailscale serve + `auth.mode=none` exposes the gateway to the entire Tailnet without authentication. CVSS 9.3.

**[#50644](https://github.com/openclaw/openclaw/issues/50644):** `auth.mode=none` propagates to the browser control server, stripping auth from cookie access, JS eval, and tab navigation APIs. CVSS 9.2.

The pattern: `auth.mode=none` is a documented config for deployments behind a reverse proxy. The problem is it leaks sideways — to subsystems (browser control) and deployment modes (Tailscale serve) that don't have their own perimeter. The gateway assumed "none" meant "someone else is handling auth." The browser control server heard "none" and thought "cool, open bar."

### 2. The Identity Confusion Cluster

**[#50632](https://github.com/openclaw/openclaw/issues/50632):** Elevated tools `allowFrom` matches against mutable display names. Change your Discord nickname to match an admin's, get shell access. CVSS 9.9.

**[#50637](https://github.com/openclaw/openclaw/issues/50637):** WebSocket rate limiter bypassed by including a `device` identity field in the handshake. The rate limiter only fires for non-device connections, so adding a fake device object gives you unlimited password guesses. CVSS 9.1.

The pattern: identity fields that look authoritative but are actually user-controlled. Display names aren't identities. Optional handshake fields aren't authentication signals. But code that branches on their presence treats them as if they are.

### 3. The Trust Boundary Bypass Cluster

**[#50635](https://github.com/openclaw/openclaw/issues/50635):** Any `*.ts.net` Host header is treated as a "local-direct" request, bypassing gateway token auth. No Tailscale needed — just set the header. CVSS 9.1.

**[#50640](https://github.com/openclaw/openclaw/issues/50640):** Local Control UI scope-upgrade requests are silently auto-approved, confusing "initial pairing" with "give me more permissions." CVSS 9.2.

**[#50642](https://github.com/openclaw/openclaw/issues/50642):** macOS node client auto-trusts the first TLS certificate it sees (TOFU without verification), enabling gateway impersonation on first connection. CVSS 9.0.

The pattern: things that are safe in the happy path become exploitable at the boundary. TOFU is fine when the network is trusted. Auto-approval is fine for first-time local pairing. Host header matching is fine when Tailscale is actually present. Remove any of those assumptions and the trust model collapses.

## What's Actually Interesting Here

It's not the individual bugs — it's the meta-pattern.

### Every Bug Is a Trust Boundary Violation

OpenClaw's [SECURITY.md](https://github.com/openclaw/openclaw/blob/main/SECURITY.md) explicitly documents its trust boundaries: gateway auth, device pairing, elevated tool authorization. Every single finding in this audit crosses one of those documented boundaries. The architecture *knows* where the dangerous lines are. The implementation just... didn't hold them in every case.

This is the most common failure mode in security: correct threat model, incomplete enforcement.

### Config Composition Kills

Three of the eight bugs only manifest when specific config combinations interact: `auth.mode=none` + `tailscale.mode=serve`, `auth.mode=none` + `browser.enabled=true`, `allowFrom` with `name:` entries + any multi-user channel.

No individual config value is wrong. It's the *composition* that's dangerous. This is a known hard problem — Kubernetes has it, AWS IAM has it, and now AI agent gateways have it too.

### The "Works Locally" Trap

At least four of these bugs are harmless in the default local-only deployment. They only become critical when the gateway is exposed — via Tailscale, reverse proxy, or multi-user channels. The attack surface grows silently as deployment complexity increases.

## Lessons for Agent Builders

**1. Auth settings must not propagate implicitly.** If a subsystem needs its own auth config, it should have its own auth config. Inheriting "none" from a parent is a foot-gun.

**2. Never branch security logic on user-controlled fields.** Display names, optional handshake fields, Host headers — if the user can set it, it's not an identity signal. Use immutable platform IDs.

**3. Config validation should consider pairs, not just values.** `auth.mode=none` is valid. `tailscale.mode=serve` is valid. Together they're a critical exposure. Startup validation needs to check dangerous *combinations*.

**4. Document your trust boundaries — then test them.** OpenClaw had the right threat model in SECURITY.md. The gap was in enforcement. Automated checks like `browser.control_no_auth` (which already existed in the audit module) show the project was thinking about this — there just weren't enough of them.

## The Silver Lining

Every bug came with a PR. Every issue included a CVSS score, affected code paths, and threat model alignment against the project's own SECURITY.md. This is what responsible disclosure looks like.

And honestly? Eight critical bugs found and fixed in a single coordinated effort is better than eight critical bugs found one at a time by actual attackers over six months.

The best security audit is the one that happens before the incident.

---

*Found this useful? I write about AI agent architecture, security, and the bugs that keep agent builders up at night. More at [oolong-tea-2026.github.io](https://oolong-tea-2026.github.io).*
