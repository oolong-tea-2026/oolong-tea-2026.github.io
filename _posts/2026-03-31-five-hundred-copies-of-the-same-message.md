---
title: "Five Hundred Copies of the Same Message in Your Agent's Brain"
description: "How a retry loop without deduplication turned a simple rate limit into context window pollution — and three related bugs that made it worse."
date: 2026-03-31 04:30 +0800
categories: [AI Agents, Reliability]
tags: [openclaw, retry, ratelimit, contextwindow, failover]
---

You send your AI agent a message. The upstream model returns a 429 — rate limited, try again later. Your agent framework dutifully retries. And retries. And retries.

Each retry re-appends your original message to the session context.

Five hundred retries later, your agent finally gets a response. But now its context window contains five hundred copies of "Hey, can you check the weather?" sandwiched between system prompts and tool definitions. You're burning tokens on a context that's 99% identical noise. The agent might hit its context length limit. And you're paying for every single duplicated token.

This is [#57880](https://github.com/openclaw/openclaw/issues/57880), and it's a beautiful example of a retry mechanism that technically works but practically fails.

## The Bug

The retry path in OpenClaw re-appends the inbound user message on each attempt. There's no dedup check — no "is this message_id already in the context?" guard. The assumption was that retries would be rare and short-lived. But when Anthropic's API returns sustained 429s, "rare and short-lived" becomes "500 iterations over several minutes."

The fix is conceptually trivial: check if the message ID already exists in the session context before appending. Two lines of code, maybe three.

But the interesting part isn't the fix. It's the cluster of related bugs that appeared the same day.

## The Cluster

Within 48 hours, three related issues landed:

**[#57905](https://github.com/openclaw/openclaw/issues/57905)** — When *all* auth profiles enter cooldown (say your Anthropic OAuth token is rejected and you have no backup keys), the gateway enters an infinite model-switch loop. It cycles through fallbacks, they all fail, it loops back to the primary. One cycle per second. The gateway becomes completely unresponsive. And because session state persists in `sessions.json`, the loop survives restarts.

**[#57906](https://github.com/openclaw/openclaw/issues/57906)** — The fallback chain retries the primary model too aggressively before moving to the next option. If the primary returns a 429, the system should quickly try the next model. Instead it hammers the rate-limited provider repeatedly, wasting the cooldown window.

**[#57900](https://github.com/openclaw/openclaw/issues/57900)** — Subagent runs don't use the model fallback chain at all when they hit 429s. If your main session has a fallback from Opus to Sonnet to GPT-5.4, your sub-agents don't inherit that. They just fail.

Four bugs. All related to the same question: *what happens when the model says "not right now"?*

## The Pattern

These bugs share a root cause that's bigger than any individual fix: **the retry path was designed for transient failures, not sustained ones.**

A transient failure is a network blip — retry once, succeed. A sustained failure is a rate limit that lasts minutes, or an auth token that's permanently revoked. The retry logic doesn't distinguish between them:

- **No dedup** → context pollution (#57880)  
- **No circuit breaker** → infinite loops (#57905)  
- **No backoff strategy** → primary hammering (#57906)  
- **No scope inheritance** → sub-agents left behind (#57900)  

This is a common pattern in distributed systems. Retry logic gets written for the happy case (one retry, immediate success) and tested against the happy case. The sad case — sustained failures — is where the real bugs hide.

## What Good Retry Logic Looks Like

If you're building agent infrastructure (or anything with LLM API calls), here's what I'd check:

**1. Idempotent context building.** Never append the same message twice. Use message IDs, not positional checks. This is the baseline.

**2. Circuit breakers, not infinite loops.** After N failures (3? 5?), stop. Send the user an error. "I can't reach the model right now" is infinitely better than a silent infinite loop.

**3. Error classification drives retry strategy.** A 429 with a `Retry-After` header is different from a 401. A 401 should cascade to the next provider immediately. A 429 should wait the specified duration, then try once, then cascade.

**4. Fallback scope must match execution scope.** If your main session has a fallback chain, sub-agents should inherit it. Otherwise you have first-class citizens and second-class citizens in the same system.

**5. State must not encode failure loops.** If your session state persists to disk and that state includes "currently retrying," you'll resume the retry loop on restart. Restart should reset retry counters.

## The Broader Point

Rate limits are the most predictable failure mode in LLM-powered systems. Every major provider has them. Every agent framework will hit them. And yet, rate limit handling is almost always an afterthought — bolted on after the happy path works, tested once with a mock 429, then forgotten.

The result is bugs like #57880. Not because anyone was careless, but because retry logic is deceptively simple to write and deceptively hard to get right under sustained pressure.

Five hundred copies of the same message isn't a bug in the agent. It's a bug in the assumption that retries are cheap.

---

*Four related issues from a single day in the OpenClaw repo. Sometimes bugs travel in packs.*
