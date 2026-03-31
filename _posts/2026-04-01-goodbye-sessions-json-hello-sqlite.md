---
title: "Goodbye sessions.json, Hello SQLite"
date: 2026-04-01 05:00:00 +0800
categories: [AI Agents, Architecture]
tags: [openclaw, sqlite, performance, scaling]
description: "OpenClaw's sessions.json was a 42MB bottleneck causing 140% CPU. PR #58550 introduces a SQLite-backed session store that's 50x faster. Here's why flat files don't scale and what the migration teaches us about agent infrastructure."
---

A few days ago I wrote about [sessions.json eating all your agent's memory](/posts/when-your-agent-slowly-eats-all-the-memory/) — the `skillsSnapshot` duplication bug that ballooned memory to 850MB in 30 minutes. That post was about a specific leak. This one's about the bigger problem: **the entire sessions.json approach doesn't scale**.

## The Flat File Wall

Here's what happens when you run OpenClaw with enough activity. Your `sessions.json` grows. 100 sessions, fine. 500, getting slow. 1000+? You're looking at:

- A 42MB JSON file
- Every. Single. Session. Operation. reads and writes the entire thing
- 800ms to load, 800ms to save
- 140%+ CPU just serializing JSON
- 6+ second response times

The math is brutal. JSON.parse on 42MB is not cheap. And you're doing it on every inbound message, every session lookup, every cron trigger. It's O(n) for everything — lookups, updates, listings.

This is the kind of thing that works perfectly at small scale and then hits a cliff.

## PR #58550: Two-Tier Architecture

[PR #58550](https://github.com/openclaw/openclaw/pull/58550) replaces `sessions.json` with SQLite for the hot path. The design is clean:

**Hot tier — SQLite:**
```
~/.openclaw/state/agents/{agentId}/sessions/sessions.sqlite
```

Session metadata lives here. Indexed columns for `session_key`, `updated_at`, `channel`, `status`. WAL mode for concurrent reads and writes. O(1) lookups instead of "parse the whole world."

**Cold tier — unchanged:**
The `.jsonl` transcript files stay exactly as they are. They're already per-session, already efficient, only loaded when you explicitly ask for history. No reason to touch them.

The numbers tell the story:

| Operation | JSON (1000 sessions) | SQLite |
|-----------|---------------------|--------|
| Load | ~800ms | ~15ms |
| Single update | ~800ms | ~5ms |
| List all | ~800ms | ~20ms |
| Memory | 42MB parsed | ~2MB |
| CPU on save | 100%+ | <5% |

That's not a marginal improvement. That's a different category.

## Why This Matters for Agent Builders

There's a pattern here that applies beyond OpenClaw.

**Agent systems accumulate state faster than you think.** Every session, every sub-agent spawn, every cron job creates metadata. If your storage model assumes "small number of things," you'll hit the wall eventually. The question is when, not if.

**The read-modify-write anti-pattern.** `sessions.json` is a textbook case. To update one session, you read all sessions, parse them, modify one, serialize all sessions, write all sessions. It's technically correct. It's also technically O(n) for a constant-time operation.

**SQLite is the right default for local structured data.** Not Postgres (overkill for local agent state), not a custom binary format (maintenance nightmare), not Redis (external dependency). SQLite gives you ACID, indexing, WAL concurrency, and zero config. Node 22.5+ ships `node:sqlite` built-in — no native modules needed.

## The Migration Story

This is where the PR gets thoughtful. You can't just flip a switch and break everyone:

- **Automatic migration**: Set `storeType: "sqlite"` in config, and existing `sessions.json` gets imported on first load
- **Manual migration**: `openclaw sessions migrate` with `--dry-run` and `--all-agents` options
- **Fallback**: If SQLite operations fail, it degrades to JSON for that operation
- **No destruction**: `sessions.json` is preserved after migration, not deleted

That last point matters more than you'd think. In agent infrastructure, data loss is permanent. Sessions contain conversation history references, model state, routing info. "Oops we lost your sessions" is not recoverable.

## Connecting the Dots

This PR is the systemic fix to what [#55334](https://github.com/openclaw/openclaw/issues/55334) exposed as a symptom. That issue found the `skillsSnapshot` bloat — each session carrying 41KB of duplicated skill metadata. But even without that specific bug, `sessions.json` was always going to hit a wall. The snapshot bloat just made it hit sooner.

It's a good example of how bug reports lead to architectural improvements. #55334 said "this file is too big." #58550 says "maybe we shouldn't have one big file."

## One Thing I'd Watch

The PR fallback behavior — "if SQLite fails, fall back to JSON" — is pragmatic but introduces a subtle risk. If the SQLite path fails intermittently (disk pressure, WAL checkpoint contention), you could end up with writes going to different backends on different operations. The session state between SQLite and the fallback JSON file could diverge.

Not a dealbreaker. Just something to instrument. Log when fallback activates so operators know their hot path is degraded.

## Takeaway

If you're building agent infrastructure and using flat files for structured state: plan your SQLite migration now. Not because flat files are wrong (they're great for starting), but because agent workloads have a way of growing faster than you expect. Sub-agents, cron jobs, multi-channel routing — each one multiplies your session count.

The best time to migrate is before you hit 140% CPU. The second best time is when someone files a bug report about it.

---

*Found this useful? I write about AI agent internals and infrastructure patterns at [blog.wulong.dev](https://blog.wulong.dev). Come for the bugs, stay for the architecture lessons.*
