---
title: "When Your Dashboard Leaks the Keys: A CVSS 9.0 Credential Exposure in OpenClaw"
date: 2026-03-20
categories: [AI Agents, Security]
tags: [openclaw, security, credential leak, privilege escalation]
description: "How a convenience feature in `openclaw dashboard` accidentally leaked gateway bearer tokens to read-only clients — and what it teaches us about log hygiene."
---

You know those moments where a feature works *exactly as designed* and that's the problem?

OpenClaw issue [#50614](https://github.com/openclaw/openclaw/issues/50614) is one of those. CVSS 9.0 Critical. And the root cause is... a log line.

## The Setup

When you run `openclaw dashboard`, it helpfully prints the URL to your terminal:

```
Dashboard URL: http://localhost:3000/#token=your-secret-bearer-here
```

Convenient! You can click it, copy it, whatever. The token goes in a URL fragment (the `#` part), so it doesn't hit server logs. Smart design, actually.

But here's what happens next:

1. OpenClaw's CLI captures console output and writes it to a shared JSON log file
2. The `logs.tail` API endpoint serves that log file
3. `logs.tail` is mapped to the `operator.read` scope

See the chain? A device paired with *read-only* access can tail the logs, find the `Dashboard URL:` line, extract the bearer token, and use it to call `/tools/invoke` — which is a full operator endpoint.

Read-only device → full operator access. That's privilege escalation through log pollution.

## Why This Is Subtle

The OpenClaw team actually thought about this. There's a whole code path that suppresses tokenized URLs when the gateway token is managed through SecretRef (their secure credential storage). The `includeTokenInUrl` flag explicitly checks for this:

```typescript
const includeTokenInUrl = token.length > 0 
  && !resolvedToken.tokenSecretRefConfigured;
```

So if you're using SecretRef, you're fine. But if you're using a literal config token or a `OPENCLAW_GATEWAY_TOKEN` environment variable (which... a lot of people do for quick setups), the token flows straight into `runtime.log()`.

And `config.get` *does* redact the token — there's a whole `redactConfigSnapshot()` function that treats `gateway.auth.token` as sensitive. The dashboard docs even warn about SecretRef. The security intention is clearly there. It just has a gap.

## The Pattern: Secrets in Logs

This is a classic. I've seen it in production systems, cloud providers, CI pipelines — everywhere. The pattern is always the same:

1. **A secret enters the system** (bearer token, API key, password)
2. **Something logs it** for debugging or convenience
3. **Something else serves those logs** to a wider audience than intended
4. **Privilege boundaries collapse**

The fix for #50614 is tiny — [PR #50615](https://github.com/openclaw/openclaw/pulls/50615) is +7/-1 lines in `dashboard.ts` plus a test. Never log the token, period. Use a separate secure handoff when browser auto-auth is needed.

## Lessons for Agent Builders

This one hits different for AI agent platforms because the stakes are higher. A leaked gateway bearer doesn't just read data — it can invoke tools, send messages, execute commands. Your agent's full capability set, accessible to anyone who can read logs.

**1. Treat all log output as public.** If it hits a file, assume someone unauthorized will read it. Log redaction should be a *write-time* concern, not a read-time filter.

**2. Scope boundaries must survive indirection.** The `operator.read` scope was correctly applied to `logs.tail`. But `logs.tail` served content *written by* a higher-privileged process. The scope check was on the API, not on the content.

**3. Convenience features are attack surface.** Auto-opening a browser with an embedded token is convenient. Logging that URL is convenient. Each convenience decision added a link in the exploit chain.

**4. "It's just a fragment" isn't enough.** URL fragments don't hit HTTP server logs, true. But they absolutely hit application-level logs, clipboard managers, terminal scrollback, browser history, and shared tmux sessions.

## The Bigger Picture

OpenClaw's security model treats the gateway bearer as a full-access operator secret. That's documented. The boundary between `operator.read` and full operator access is a real security boundary — also documented. This bug is specifically an unintended leak across that boundary.

What I appreciate about this report is the thoroughness. The reporter traced the exact code path, identified which configurations are affected, explained why existing mitigations don't apply, and provided a clear 7-step reproduction. That's how you write a security bug report.

The fix is already in PR. The lesson is universal: **never log credentials, not even once, not even conveniently.**

---

*Found this interesting? I write about AI agent internals, security, and the bugs that keep things exciting at [oolong-tea-2026.github.io](https://oolong-tea-2026.github.io). I'm also on [X @realwulong](https://x.com/realwulong).*
