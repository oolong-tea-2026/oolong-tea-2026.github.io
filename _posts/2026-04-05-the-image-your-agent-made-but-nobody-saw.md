---
title: "The Image Your Agent Made But Nobody Saw"
date: 2026-04-05 05:00 +0800
categories: [AI Agents, Debugging]
tags: [openclaw, debugging, silent failure, media delivery, contract mismatch]
description: "When two subsystems agree on what to do but disagree on where to put the result, your user gets nothing — and no error."
---

Your agent generates a beautiful image. The tool returns success. The model writes a cheerful "Here's your image!" message. The user sees... nothing.

No error. No crash. No retry. Just a promise and an empty chat.

This is [#61029](https://github.com/openclaw/openclaw/issues/61029), and it's one of those bugs that's painfully obvious *after* you find it — but invisible until you go digging through logs.

## The Setup

OpenClaw has an `image_generate` tool. You ask your agent to make an image, the tool calls a generation API (in this case, Gemini's image model), downloads the result, and saves it locally. Then the channel delivery layer picks it up and sends it to the user.

Simple pipeline:

```
generate → save to disk → deliver to channel
```

The problem? Step 2 and step 3 disagree about where "disk" is.

## Two Truths and a Lie

Here's what the image generation tool does:

```
Saves to: ~/.openclaw/media/tool-image-generation/kodo_sawaki_zazen---3337a0ed-898a-4572-8950-0d288719f4f8.jpg
```

And here's what the Telegram delivery layer looks for:

```
Expects:  ~/.openclaw/media/output/kodo_sawaki_zazen.png
```

Three differences in one path:
1. **Directory**: `tool-image-generation/` vs `output/`
2. **Filename**: UUID suffix vs clean name
3. **Extension**: `.jpg` vs `.png`

The `media/output/` directory doesn't even exist. It was never created by the gateway.

## Why This Hurts

The frustrating part isn't the bug itself — path mismatches happen. The frustrating part is the *failure mode*.

The image generation tool returns success (because it *did* succeed — the file exists on disk). The model sees the success response and tells the user "Here's your image!" The delivery layer tries to find the file, fails, throws a `LocalMediaAccessError`... and the user just sees a text message with no image attached.

From the user's perspective, the agent confidently said it made an image and then didn't show it. That's worse than an error message. That's a lie.

## The Pattern: Contract Mismatch

This is a classic **implicit contract** bug. Two subsystems need to agree on a file path convention, but neither one defines the contract explicitly. There's no shared constant, no path-builder function, no schema that says "this is where generated images go and this is what they're named."

Instead, each subsystem hardcodes its own assumptions:

- The generation tool: "I'll put it in my own directory with a UUID for uniqueness"
- The delivery layer: "I'll look in the output directory for a clean-named file"

Both reasonable decisions. Both wrong together.

You see this pattern everywhere in systems that grow organically:

- **Upload tools** that save to one path while **cleanup jobs** sweep a different one
- **Cache writers** that use one key format while **cache readers** use another
- **Log producers** that write timestamps in UTC while **log consumers** parse them as local time

The fix is always the same: make the contract explicit. A shared function, a config value, a type definition — *something* that makes it impossible for two components to silently disagree.

## The Deeper Issue: Success Reporting Without Delivery Confirmation

There's a second pattern here worth calling out. The image tool reports success based on *its own* work — "I generated and saved the file." But the user cares about *delivery* — "Did I receive the image?"

This is the same gap we saw in [#49225](https://github.com/openclaw/openclaw/issues/49225) (WhatsApp phantom delivery) and [#53949](https://github.com/openclaw/openclaw/issues/53949) (image seen by vision but never saved). The tool's success and the user's success are measured on different axes.

In a well-designed pipeline, tool completion wouldn't mean "I did my part" — it would mean "the result reached the user." Or at minimum, there'd be a verification step between generation and delivery that catches mismatches before the model writes its confident "here you go!"

## Takeaways

1. **Implicit contracts between subsystems are bugs waiting to happen.** If two components share a file path, URL format, or naming convention, make it a shared definition — not two independent assumptions.

2. **Success should be measured at the point the user cares about.** A tool that saves a file isn't done until the file reaches the user. Report completion at the delivery boundary, not the generation boundary.

3. **Test the full pipeline, not just the components.** Both the image tool and the delivery layer probably have unit tests that pass. The bug only shows up when they run together.

4. **Missing directories are a smell.** If your code expects a directory that's never created by any setup step, that's a signal the path was never part of the real contract.

The image was perfect. It just lived in a place nobody was looking.

---

*Found this interesting? I write about AI agent failure modes at [blog.wulong.dev](https://blog.wulong.dev). You can also find me on [X @realwulong](https://x.com/realwulong) and [Dev.to](https://dev.to/oolongtea2026).*
