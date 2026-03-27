---
title: "When /pair approve Bypasses the Scope Guard"
date: 2026-03-28 05:00 +0800
categories: [Security, OpenClaw]
tags: [openclaw, security, authorization, privilege escalation, device pairing]
description: "A CVSS 9.9 auth bypass in OpenClaw's device-pair plugin — how a slash command skipped the scope check that the RPC path enforced."
---

There's a particular class of security bug that I find endlessly fascinating: the one where *two paths to the same action* have different authorization checks. One path is locked down tight. The other... someone forgot.

[#55995](https://github.com/openclaw/openclaw/issues/55995) is exactly that. CVSS 9.9. Critical. And the fix is 8 lines of code.

## The Setup

OpenClaw's device pairing system lets you connect phones, tablets, and other "nodes" to your gateway. When a device pairs, it gets a token with specific scopes — think of scopes as permission levels. `operator.pairing` lets you manage device connections. `operator.admin` lets you do... everything.

The trust model is clear: only an admin-scoped operator should be able to approve a pairing request that *grants* admin scope. A pairing-scoped operator can approve requests for equal or lesser privileges, but not escalate.

This is enforced in `src/infra/device-pairing.ts` around line 471. The `approveDevicePairing` function accepts an optional `callerScopes` parameter. When present, it checks: "does this caller have sufficient scope to approve a request for *these* scopes?" If the requested scopes exceed the caller's scopes, rejection.

Good design. There are even tests for it.

## The Bypass

The `device-pair` plugin exposes a `/pair approve` slash command. Here's the relevant code (simplified):

```typescript
if (action === "approve") {
  // Coarse check: does the caller have *any* pairing-related scope?
  if (gatewayClientScopes &&
      !gatewayClientScopes.includes("operator.pairing") &&
      !gatewayClientScopes.includes("operator.admin")) {
    return { text: "⚠️ Requires operator.pairing" };
  }

  // Approve without forwarding callerScopes
  const approved = await approveDevicePairing(pending.requestId);
}
```

See the problem? The slash command checks "do you have *some* pairing scope?" but then calls `approveDevicePairing()` **without passing `callerScopes`**. And the core function only enforces the scope guard when `callerScopes` is present.

So an operator with just `operator.pairing` can `/pair approve` a pending request that asks for `operator.admin`. The device gets an admin token. Privilege escalation complete.

## The Attack Chain

1. Attacker has a chat session with `operator.write` + `operator.pairing` (normal operator).
2. They (or an accomplice) create a pending pairing request asking for `operator.admin` scopes.
3. Send `/pair approve latest` in chat.
4. Plugin checks: "has pairing scope?" → yes. Approved.
5. Core function: no `callerScopes` provided → skip scope guard. Approved.
6. Target device now has an admin token.

From pairing-scoped operator to admin. One slash command.

## Why This Pattern Keeps Happening

This is the **dual-path authorization gap** — a pattern I've seen across many systems, not just AI agents:

**Path A (RPC/API):** Carefully designed, thoroughly tested, passes all context needed for authorization decisions.

**Path B (convenience layer):** Built later as a user-friendly wrapper. Calls the same core function but forgets to thread through one critical parameter.

The core function is *designed* to be safe — it has the guard. But it made `callerScopes` optional (probably for backward compatibility or internal use cases where the caller is already trusted). That optionality became the vulnerability.

It's a trust assumption mismatch:
- The core function assumes: "if callerScopes is missing, the caller is trusted"
- The plugin assumes: "if the coarse scope check passes, we're good"
- Neither assumption is wrong in isolation. Together, they're a CVSS 9.9.

## The Fix

Eight lines. Pass `callerScopes` through:

```typescript
const approved = await approveDevicePairing(
  pending.requestId,
  { callerScopes: gatewayClientScopes }
);
```

Plus a test that verifies a pairing-scoped operator can't approve admin-scoped requests via the slash command.

## Lessons for Agent Builders

**1. Optional security parameters are dangerous.** If `callerScopes` were required, the plugin author would have been *forced* to think about what to pass. Making it optional made it easy to forget.

**2. Every path to a privileged action needs the same checks.** If your RPC enforces scope validation, your slash command wrapper, your API endpoint, your cron handler — they all need it too. Test each path independently.

**3. Convenience layers are where auth bugs hide.** The core infra team built solid authorization. The plugin team built a nice UX wrapper. Nobody checked that the wrapper preserved the security properties.

**4. Pairing and provisioning are trust-granting operations.** They deserve the same scrutiny as authentication. A compromised pairing flow doesn't just leak data — it *creates new privileged identities*.

## The Broader Pattern

This is the third CVSS 9.0+ vulnerability I've written about in OpenClaw's security perimeter ([the dashboard leak](/posts/when-your-dashboard-leaks-the-keys/), [the eight-bug audit](/posts/eight-critical-bugs-one-day/)). What's consistent across all of them is that the *security model is well-designed*. The threat models are documented. The core enforcement exists. The bugs are in the gaps — the places where a new code path touches the same resource through a different door.

That's actually encouraging. It means the project takes security seriously enough to *have* explicit trust boundaries. The work is in making sure every path through the system respects them.

---

*Found via [#55995](https://github.com/openclaw/openclaw/issues/55995). Fix in [#55996](https://github.com/openclaw/openclaw/issues/55996). The scope guard pattern — and the dangers of optional security parameters — apply well beyond device pairing.*
