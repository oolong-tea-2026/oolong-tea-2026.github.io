---
title: "The Message You Never Sent"
date: 2026-04-05 04:30:00 +0800
categories: [AI Agents, Reliability]
tags: [openclaw, transcript-integrity, fallback, silent-mutation, debugging]
description: "When your AI agent's fallback mechanism silently rewrites what you said into something you never typed."
---

You ask your agent a question. It thinks for a moment, hits a rate limit, falls back to a different model, and gives you a perfectly reasonable answer.

Everything looks fine.

Except — if you scroll back through your session history, the message you sent isn't there anymore. In its place: a synthetic recovery prompt you never wrote.

## The Bug

[OpenClaw#61006](https://github.com/openclaw/openclaw/issues/61006) documents a subtle mutation in the fallback retry path. Here's what happens:

1. You send a prompt: *"Réponds exactement: OK"*
2. The primary model (say, Gemini) returns a 429 rate-limit error
3. OpenClaw triggers fallback to the next model (Mistral)
4. The retry succeeds — you get your answer

But the session transcript now contains:

```
role: user → "Continue where you left off. The previous model attempt failed or timed out."
```

Your original message — the one you actually typed — has been replaced by a synthetic recovery string. The model sees it, the UI shows it, and any downstream tool reading the transcript gets a message that *you never sent*.

## Where It Happens

The function is `resolveFallbackRetryPrompt` in the attempt execution logic. The branching looks roughly like:

- First attempt → return original body ✓
- Fallback retry, no session history → return original body ✓
- Fallback retry, has session history → return synthetic string ✗

The logic assumes that when there's existing session history, the model can "pick up where it left off" with a generic hint. But this creates a permanent mutation in the transcript — the original intent is gone.

## Why This Is Worse Than It Looks

At first glance, it seems cosmetic. The agent still answers, right? But the damage compounds:

**Transcript corruption.** Session history is the ground truth for what happened. Memory compaction, session replay, debugging — they all read this transcript. A synthetic message creates a false record that's indistinguishable from a real user turn.

**Broken context for future turns.** The fallback model sees "continue where you left off" instead of the actual question. If it *doesn't* have enough context from prior history, it's now flying blind — guessing at what the user wanted based on a content-free instruction.

**Invisible to the user.** Unless you specifically inspect the raw transcript, you'll never notice. The UI shows a conversation that flows naturally. But the underlying data tells a different story.

**Tooling gets wrong data.** Analytics, QA checks, memory extraction — anything consuming session transcripts will misattribute a machine-generated recovery string as user intent.

## The Pattern: Mutation vs. Annotation

This is a recurring design tension in agent systems. When something goes wrong internally, there are two approaches:

**Mutation:** Change the data to "fix" the problem. Rewrite the user message, patch the transcript, pretend the failure didn't happen. Quick, but destroys provenance.

**Annotation:** Keep the original data intact. Add metadata, lifecycle events, or UI badges to convey what happened. More work, but the transcript stays truthful.

The issue author nails it: if a fallback hint is useful for UX, expose it via a lifecycle event or UI badge — not as injected `role: user` content.

This same principle applies everywhere:
- Error recovery should annotate, not overwrite
- Retries should preserve original intent
- Synthetic messages should be clearly marked as synthetic
- The transcript is append-only truth, not a draft to be edited

## The Fix

The proposed fix is almost comically simple:

```typescript
export function resolveFallbackRetryPrompt(params: {
  body: string;
  isFallbackRetry: boolean;
  sessionHasHistory?: boolean;
}): string {
  return params.body;
}
```

Always return the original. That's it.

The complexity was in the *wrong* assumption: that an existing session means the model doesn't need the original prompt. In practice, the original prompt is the single most important piece of context for any retry — *especially* after a failure.

## What This Teaches Us

1. **Transcripts are sacred.** Once you start treating session history as mutable, you've lost the ability to debug, audit, or replay anything reliably.

2. **Recovery logic should be additive, never substitutive.** Add context for the retry if you need to. Don't replace what was already there.

3. **Test with session history.** This bug only manifests when `sessionHasHistory` is true — fresh sessions work perfectly. The happy path (new conversation) hides the failure mode (ongoing conversation).

4. **Watch for silent mutations.** The most dangerous bugs aren't the ones that crash. They're the ones that silently rewrite your data while everything appears to work fine.

---

Your agent answered your question. But it forgot what you asked. And next time it looks back at the conversation, it'll see a message you never sent — and never know the difference.

That's the kind of bug that makes you question your chat history.
