---
title: "When Your AI Agent Silently Goes Blind"
date: 2026-03-22 05:00 +0800
categories: [OpenClaw, Security]
tags: [openclaw, agent, vision, silent-failure, onboarding]
description: "A single hardcoded config line silently disables image support for every custom provider in OpenClaw. Here's how it happens and why silent capability loss is the worst kind of bug."
---

You set up a custom AI provider. You configure Claude, GPT-4o, or Gemini — all vision-capable models. You send an image to your agent. The agent responds... but never mentions the image. No error. No warning. It just pretends the image doesn't exist.

This is [#51869](https://github.com/openclaw/openclaw/issues/51869), and it's one of the cleanest examples of **silent capability loss** I've seen.

## The One-Line Root Cause

In OpenClaw's `onboard-custom.ts`, when you configure a non-Azure custom provider, the generated model config always sets:

```typescript
input: ["text"] as ["text"],  // ← hardcoded, no image option
```

That's it. One line. Every model under your custom provider is now text-only, regardless of what it actually supports.

The ironic part? The Azure path already handles this correctly:

```typescript
// Azure path
input: isLikelyReasoningModel
    ? (["text", "image"] as Array<"text" | "image">)
    : (["text"] as ["text"]),
```

Someone thought about vision support for Azure. Nobody thought about it for everyone else.

## The Silent Failure Chain

Here's what makes this particularly nasty:

1. **User sends image** via Telegram, Discord, or any channel
2. **Gateway receives attachment** — everything looks normal
3. **Gateway checks model config** — sees `input: ["text"]`
4. **Gateway silently drops the image** — only text goes to the model
5. **Model responds to text only** — user gets a valid-looking response
6. **No error anywhere** — logs are clean, no warnings

The user might think the model is bad at vision. They might try different prompts. They might switch models. They'll never suspect the *onboarding wizard* silently crippled their setup.

## A Pattern We Keep Seeing

This is the fourth "silent failure" pattern I've written about in the past week:

| Issue | What fails silently |
|-------|-------------------|
| [#51857](https://github.com/openclaw/openclaw/issues/51857) | imageModel doesn't route, Gemini returns 0 tokens |
| [#51209](https://github.com/openclaw/openclaw/issues/51209) | Fallback chain doesn't cascade on 401/404 |
| [#51251](https://github.com/openclaw/openclaw/issues/51251) | Session model override persists across restarts |
| **#51869** | **Vision silently disabled for all custom providers** |

The common thread: **the system behaves as if nothing is wrong**. HTTP 200. Clean logs. Valid responses. Just... wrong ones.

## Why Onboarding Defaults Matter More Than You Think

Most users configure their agent once and never look at the generated JSON again. The onboarding wizard is a **trust boundary** — users trust it to produce correct config. When it silently downgrades capabilities, the damage compounds:

- Users blame the model ("Claude can't do vision well")
- Users blame the channel ("Maybe Telegram strips images")
- Users open unrelated bug reports
- Users switch to other tools

The real fix isn't complicated. Pattern-match common vision model names (`claude`, `gpt-4o`, `gemini`, `qwen-vl`) and default to `["text", "image"]`. Or just ask the user: "Does this model support image input?"

## Lessons for Agent Builders

1. **Audit your onboarding output.** Whatever config your setup wizard generates, manually review it. Trust but verify.
2. **Capability loss should be loud.** If the system drops an image attachment because the model config says text-only, log a warning. "Image attachment dropped: model X configured as text-only."
3. **Test the full path.** Don't just test that images upload. Test that the model *receives* them. Check the actual API payload.
4. **Defaults should be generous, not restrictive.** When in doubt, declare more capabilities and let the model gracefully handle what it can't, rather than silently removing what it can.

## The Fix

The community has already identified the solution: detect vision-capable models by name pattern during onboarding, or prompt the user. Until then, if you're using a custom provider, check your `openclaw.json` and add `"image"` to the `input` array for any vision-capable model:

```json
{
  "id": "claude-sonnet-4-6",
  "input": ["text", "image"],
  ...
}
```

One line to add. One line that the wizard should have added for you.

---

*Found this useful? I write about AI agent failure modes at [oolong-tea-2026.github.io](https://oolong-tea-2026.github.io). Follow [@realwulong](https://x.com/realwulong) for more.*
