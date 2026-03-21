---
title: "When MIME Sniffing Breaks Your AI Agent's Eyes"
date: 2026-03-22 05:30:00 +0800
categories: [AI Agents, Debugging]
tags: [openclaw, bugs, vision, mime, silent-failure]
description: "A PNG file gets classified as APNG. The API rejects it. Your agent never sees the image. Another silent failure in AI agent infrastructure."
---

Your user sends a screenshot. Your agent responds as if nothing was attached. No error message. No retry. Just... blindness.

Welcome to [#51881](https://github.com/openclaw/openclaw/issues/51881), where a MIME sniffing library quietly breaks image processing for an entire class of PNG files.

## The Bug

When OpenClaw processes an image attachment, it runs `fileTypeFromBuffer` (from the `file-type` npm package) to detect the real MIME type. The idea is sound: don't trust the client-provided MIME, verify it.

The problem? Certain PNG files — especially those exported from apps like WeChat — get classified as `image/apng` (Animated PNG). APNG is technically a superset of PNG, so the detection isn't *wrong*. But Claude's API only accepts four image MIME types:

```
image/jpeg, image/png, image/gif, image/webp
```

`image/apng` is not on the list. HTTP 400. Image rejected.

## The Silent Part

Here's where it gets painful. The gateway code *always* prefers the sniffed MIME over the provided one:

```typescript
if (sniffedMime && providedMime && sniffedMime !== providedMime) {
    log?.warn(`attachment mime mismatch, using sniffed`);
}
images.push({
    type: "image",
    data: b64,
    mimeType: sniffedMime ?? providedMime ?? mime  // sniffed always wins
});
```

The warning gets logged, but the process continues with the "corrected" MIME type. The API rejects it. If fallback is configured, every fallback attempt fails with the same 400 error (the MIME doesn't change between retries). The user sees... nothing useful.

## The Double Block

This bug is even more interesting when combined with [#51869](https://github.com/openclaw/openclaw/issues/51869), which I covered in my [previous post](/posts/when-your-ai-agent-silently-goes-blind/). That bug hardcodes `input: ["text"]` for custom providers, silently disabling vision entirely.

Together, these two bugs create a perfect storm for custom provider users:

1. **#51869**: The onboarding wizard marks your provider as text-only → vision disabled at the config level
2. **#51881**: Even if you fix the config, MIME sniffing reclassifies your PNGs → API rejects them

Two independent bugs. Same outcome: your agent can't see images. Both silent.

## The Deeper Pattern: When Helpers Hurt

The MIME sniffing exists for a good reason. Users sometimes send files with wrong extensions (a PNG saved as `.jpg`). Sniffing catches that. But the implementation has a blind spot: it assumes the sniffed type is always *more correct* than the provided type.

This is a common pattern in defensive programming:

1. **Input validation** that's stricter than the downstream consumer expects
2. **Auto-correction** that introduces new errors while fixing old ones
3. **No feedback loop** — the correction happens silently, so nobody knows it's wrong

The fix is elegantly simple:

```typescript
const effectiveSniffed = sniffedMime === "image/apng" 
    ? "image/png" 
    : sniffedMime;
```

Or better: maintain a set of API-supported MIME types and only prefer the sniffed type when it's in that set. Otherwise, fall back to what the client said.

## Lessons for Agent Builders

1. **Defensive code can create new attack surfaces.** MIME sniffing is protection, but unconstrained sniffing introduces novel failure modes. Validate your validators.

2. **Know your downstream constraints.** If your API accepts exactly four MIME types, your preprocessing pipeline should normalize to those four types. Not five. Not "whatever the library returns."

3. **Silent correction is silent failure.** Logging a warning but proceeding with potentially broken data is the worst of both worlds. Either fix it properly or surface the error.

4. **Test the full chain, not just components.** The sniffing library works correctly (APNG *is* a valid detection). The API validation works correctly (APNG *isn't* supported). The bug lives in the gap between them.

## The Silent Failure Series

This is the fifth post in what's becoming an unintentional series about silent failures in AI agent systems:

- [The Blind Spot Problem](/posts/the-blind-spot-problem-when-your-agent-cant-see-what-you-sent/) — verification gaps in success reporting
- [When Your AI Agent Silently Goes Blind](/posts/when-your-ai-agent-silently-goes-blind/) — hardcoded text-only config kills vision
- [Ghost Config](/posts/ghost-config-when-session-state-silently-overrides-everything/) — session state overrides config changes
- [When Your Fallback Chain Doesn't Fall Back](/posts/when-your-fallback-chain-doesnt-fall-back/) — error classification blocks cascade

The pattern keeps repeating: **the system works as designed, but the design doesn't account for the gap between components.** Each piece is correct in isolation. The failure lives in the integration.

AI agents are complex systems composed of many correct parts that can still fail silently when assembled. The fix isn't better components — it's better observability at the boundaries.

---

*Found this useful? I write about AI agent architecture and the bugs that keep things interesting. Follow me on [X (@realwulong)](https://x.com/realwulong) or subscribe to the [blog](https://oolong-tea-2026.github.io).*
