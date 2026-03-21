---
title: "The Blind Spot Problem: When Your Agent Reports Success But Processes Nothing"
date: 2026-03-22 04:30:00 +0800
categories: [AI Agents, Reliability]
tags: [openclaw, observability, silent-failures, media, verification]
description: "Four real failures where AI agents returned 200 OK while silently dropping images, files, and token counts. The verification gap is more dangerous than crashes."
---

Your agent said it processed the image. It didn't.

Your agent reported zero tokens used. It used plenty.

Your agent received the file attachment. Except it was empty.

And in every case — no error. No warning. HTTP 200 across the board.

## The Pattern Nobody's Talking About

[#51857](https://github.com/openclaw/openclaw/issues/51857) pulls together four separate community reports into a single uncomfortable observation: OpenClaw (and arguably most agent frameworks) verify that operations were *attempted*, not that they *produced correct results*.

This is a verification gap. And it's worse than crashes because crashes are loud.

## Four Failures, One Shape

### 1. The Image Model That Never Fires

You configure `imageModel` as your vision fallback. Your primary model can't handle images, so OpenClaw should route image inputs to the image model.

It doesn't. Instead, it silently falls back to the `read` tool — which tries to parse a JPEG as text. The config check passed. The routing logic decided it had a valid path. The *wrong* path.

The operator did everything right. They set the config. They tested text. It worked. They sent an image. It "worked" — meaning it didn't crash. But the image model they configured? Never touched.

### 2. Zero Tokens, Full Responses

Every Gemini API call succeeds. Responses come back fine. But `usageMetadata` from Google's response format never maps to OpenClaw's internal `usage.input/output` fields.

Result: every single Gemini call records 0 tokens. Your billing dashboard says you're using nothing. Your context tracking thinks you have infinite room. Your audit logs are structurally wrong — and have been since the integration went live.

The dangerous part? Everything *works*. You only discover this when you audit the numbers and wonder why your Gemini costs don't match your usage metrics.

### 3. Tool Call ID Collisions in Group Chats

`moonshot/kimi-k2.5` returns HTTP 400 for "duplicated tool call ID" — but only in group chats. DMs work perfectly.

Why? The tool call ID counter is session-scoped. In DMs, one user = one session = unique IDs. In group chats, multiple participants share session context, and the counter generates collisions.

This is a beautiful edge case because it's *architecturally correct* for the simple case and *architecturally broken* for the complex one. The kind of bug that passes every test you write until someone uses it in a group.

### 4. Empty Attachments From MS Teams

PDFs sent through MS Teams channels never reach the agent. The Graph API permissions? Valid. The token? Valid. The HTTP request? Returns 200.

But `downloadMSTeamsGraphMedia` returns `{ media: [] }` every time. The agent processes an empty array and moves on. No error. No "hey, I didn't get your file." Just... silence.

## Why This Matters More Than Crashes

Here's the thing about crashes: they're embarrassing but honest. Your agent throws an error, you see the stack trace, you fix it. The feedback loop is tight.

Silent success with wrong content has no feedback loop. The agent believes it succeeded. The user might not notice for hours, days, or ever. The failure compounds — wrong token counts lead to wrong context management, missed images lead to wrong conclusions, empty files lead to hallucinated analysis of nothing.

It's the [boiling frog](https://en.wikipedia.org/wiki/Boiling_frog) of AI agent reliability.

## The Verification Gap

Most agent frameworks (not just OpenClaw) check at the HTTP layer:
- Did the request succeed? ✅
- Did we get a response? ✅
- Is the status code 2xx? ✅

What they don't check:
- Does the response contain the *content type* we expected?
- Is the content *non-empty* and *well-formed*?
- Do the metadata fields (tokens, dimensions, byte counts) make sense?
- Does the actual result match the *intent* of the operation?

That last one is the hardest. "Did the image model actually process an image?" requires understanding the *purpose* of the call, not just its outcome.

## What Agent Builders Should Do

**1. Verify content, not just status.** After any media operation, check that you got bytes. After token counting, check that the count is non-zero. After file download, check that the content isn't empty. These are trivial assertions that catch non-trivial bugs.

**2. Log routing decisions, not just results.** When the system chooses which model to route to, or which fallback to use, log that decision explicitly. "Routed image to read tool because imageModel was null" is a debug message that saves hours.

**3. Treat zero as suspicious.** Zero tokens, zero bytes, zero results — these should trigger warnings in production, not silent acceptance. Zero is rarely the right answer for operations that are supposed to produce something.

**4. Test the group chat path.** Session-scoped state + multi-user contexts = collision territory. If your agent works in DMs but you haven't tested groups, you have untested code in production.

**5. Build verification into the agent loop.** Not as a separate monitoring system — as part of the agent's own reasoning. "I processed an image" should include "and I can describe what was in it." If the agent can't verify its own work, neither can you.

## The Uncomfortable Question

How many of these silent failures are happening right now, in production agents, that nobody has noticed?

The answer is: more than you think. Because the defining characteristic of this failure mode is that everything looks fine from the outside.

---

*The Driftnet project (#51857) is doing great work cataloging these patterns across the community. If you're building agents in production, their failure taxonomy is worth following.*

*— Wu Long ([@realwulong](https://x.com/realwulong))*
