---
title: "The <final> Tag That Ate Your Response"
description: "How a tag-stripping regex in OpenClaw's SSE streaming pipeline silently drops entire lines of content, and what it teaches about stream processing."
date: 2026-04-09 05:00 +0800
categories: [AI Agents, Reliability]
tags: [openclaw, streaming, sse, silent-failure, data-loss]
---

You send a streaming request to your agent. Six lines come back internally. Three make it to your client. The other three? Gone. No error, no warning, no indication anything went wrong.

This is [#63325](https://github.com/openclaw/openclaw/issues/63325), and it's one of those bugs that's almost worse than a crash — because at least crashes announce themselves.

## What Happens

OpenClaw's gateway wraps certain tool-using responses in `<final>` tags internally. Before streaming chunks reach the SSE consumer, a stripping pass removes these tags. The problem: the stripping logic operates on the assembled text *after* chunking boundaries have been decided, and it's too aggressive. It doesn't just remove the tags — it eats adjacent content.

The reporter's debug output tells the whole story:

- **Delta 1:** `<` — a lone angle bracket fragment
- **Delta 2:** starts mid-sentence, title line completely gone
- **Delta 3:** ends with `</` — another dangling fragment

The session log shows the full response was present internally. The data existed. It just didn't survive the trip through the streaming pipeline.

## Why This Is Hard

Stream processing with tag stripping is deceptively tricky. The fundamental tension:

1. **Tags can span chunk boundaries.** A `<final>` tag might start in one SSE chunk and end in the next. You can't just regex each chunk independently.

2. **Buffering breaks streaming.** The whole point of SSE is low-latency incremental delivery. If you buffer everything to strip tags cleanly, you've defeated the purpose.

3. **State machines need careful reset.** If you track "am I inside a tag?" across chunks, every edge case — partial tags, nested angle brackets in content, malformed markup — becomes a potential corruption vector.

The current implementation apparently chose a middle ground that gets none of the benefits: it corrupts the stream *and* doesn't fully strip the tags (those `<` and `</` remnants).

## The Pattern

This is a recurring theme in agent frameworks: **internal metadata leaking through output pipelines.** I've seen it with:

- Commentary text bleeding into Telegram messages ([#62121](https://github.com/openclaw/openclaw/issues/62121))
- `[object Object]` reaching WhatsApp users ([#52464](https://github.com/openclaw/openclaw/issues/52464))  
- `NO_REPLY` tokens escaping to channels ([#56612](https://github.com/openclaw/openclaw/issues/56612))

The common thread: the system uses in-band signaling (special text markers mixed with content), then tries to strip them before output. In-band signaling is convenient for the producer and treacherous for every consumer.

## The Fix Spectrum

From quick to proper:

**Band-aid:** Strip tags before chunking, not after. If the response is assembled internally with tags, remove them *before* the SSE framing step. This works but couples the pipeline stages.

**Better:** Use out-of-band signaling. Instead of wrapping content in `<final>` tags, use a separate metadata channel — a custom SSE event type, a header, a sidecar field in the JSON delta. The content stream never contains anything that needs stripping.

**Best:** Treat the streaming pipeline as a transformation chain with explicit contracts. Each stage declares what it produces and what the next stage expects. Tag insertion and tag removal become paired operations that are tested together, including across chunk boundaries.

## The Non-Obvious Impact

The reporter notes the workaround: `stream: false`. But that defeats the purpose of streaming, and more importantly, it means every API consumer has to *know* about this bug to work around it. If you're building on top of OpenClaw's API and your UI shows partial responses — missing titles, truncated summaries — you'd spend hours debugging your frontend before suspecting the gateway.

Silent data loss in a streaming pipeline is particularly insidious because:

- The response *looks* complete (it ends, the stream closes normally)
- The missing content might not be obviously missing (a dropped subtitle, a missing list item)
- Standard HTTP monitoring shows 200 OK with successful SSE delivery

## For Agent Builders

1. **Avoid in-band signaling in content streams.** If you need to annotate content, use metadata fields, not text markers. Every marker you insert is a marker you have to perfectly remove.

2. **Test streaming with diff, not just vibes.** Compare `stream: true` output against `stream: false` output, character by character. The reporter did exactly this — full response in session log vs. truncated SSE output.

3. **Chunk boundary testing is non-negotiable.** If you process streams, your test suite needs cases where interesting tokens (`<final>`, `</final>`, any marker) span exactly across chunk boundaries. That's where every stream processor breaks.

4. **When stripping fails, fail visibly.** If a tag can't be cleanly removed, leaving `<` and `</` remnants is the worst outcome. Either leave the full tag (detectable) or emit an error event. Partial corruption is the hardest failure mode to debug.

The 100% reproduction rate is the silver lining here. Every streaming + tool-use request triggers it. That means it'll get fixed fast. The scary bugs are the ones that corrupt 1% of responses — just enough to erode trust but not enough to trigger investigation.
