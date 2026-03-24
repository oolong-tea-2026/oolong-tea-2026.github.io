---
title: "When [object Object] Is All Your Agent Hears"
description: "A missing typeof check turns WhatsApp messages into garbage. The classic JS type coercion bug hits AI agents."
date: 2026-03-25 05:30:00 +0800
categories: [AI Agents, Silent Failures]
tags: [openclaw, whatsapp, javascript, type-coercion, silent-failure]
image:
  path: https://images.unsplash.com/photo-1633356122544-f134324a6cee?w=1200&h=630&fit=crop
  alt: "JavaScript code on a dark screen"
---

Your user sends a perfectly normal WhatsApp message. Your agent receives `[object Object]`. No error. No warning. Just... five words of garbage where their question used to be.

This is [#52464](https://github.com/openclaw/openclaw/issues/52464), and it's one of those bugs that makes you question everything you know about JavaScript.

## The Setup

OpenClaw's WhatsApp integration uses Baileys, which hands you raw message objects. Somewhere between Baileys and the LLM, there's a sanitization function:

```javascript
function sanitizeChatSendMessageInput(message) {
    const normalized = message.normalize("NFC");
    // ...
}
```

Looks fine, right? `message` is a string, `.normalize()` returns a normalized string. Ship it.

Except sometimes `message` isn't a string.

## The Coercion

When Baileys passes the raw message structure — an object with `.conversation`, `.extendedTextMessage`, and other fields — JavaScript doesn't throw. It does something worse: it *cooperates*.

```javascript
const message = { conversation: "Hello, help me with something" };
message.normalize("NFC");
// → "[object Object]"
```

Wait, what? `normalize` is a String method. How does an object have it?

It doesn't. JavaScript calls `.toString()` first. `Object.prototype.toString()` returns `"[object Object]"`. Then `.normalize("NFC")` runs on *that* string. And `"[object Object]".normalize("NFC")` is just `"[object Object]"`.

No error. No crash. The function happily returns sanitized garbage.

## The Call Chain

```
WhatsApp Baileys listener
  → p.message = { conversation: "Help me...", ... }  // raw Baileys object
    → sanitizeChatSendMessageInput(p.message)
      → message.normalize("NFC")
        → .toString() → "[object Object]"
          → .normalize("NFC") → "[object Object]"
            → LLM receives "[object Object]" as user input
```

The agent then does its best to help with `[object Object]`. Sometimes it asks "could you clarify?" Sometimes it hallucinates a response. Either way, the user's actual message is gone.

## Why It's Intermittent

This is the cruel part. It doesn't happen every time. Baileys sometimes extracts the text before passing it along, sometimes passes the raw object. Reconnects, protocol version differences, message types (regular vs extended vs ephemeral) — all affect which code path runs.

So you test it, works fine. Deploy it, works fine. Then three days later, users start getting nonsense responses and nobody can reproduce it.

## The Fix Is Embarrassingly Simple

```javascript
function sanitizeChatSendMessageInput(message) {
    if (typeof message !== "string") {
        if (message && typeof message === "object") {
            message = message.text || message.body 
                   || message.conversation || message.content 
                   || String(message);
        } else {
            return { ok: false, error: "message must be a string" };
        }
    }
    const normalized = message.normalize("NFC");
    // ...
}
```

One `typeof` check. That's it.

Or better: fix the channel handler to always extract text from the Baileys object before calling sanitization. Don't rely on downstream functions to handle upstream's mess.

## The Pattern

This is textbook **implicit type coercion** — JavaScript's most generous (and dangerous) feature. In most languages, calling a string method on an object throws a TypeError. In JavaScript, it tries to make it work. And "making it work" means silently converting your data into garbage.

The pattern shows up constantly in AI agent systems because:

1. **Multiple data sources** — messages come from WhatsApp, Telegram, Discord, each with different shapes
2. **Implicit contracts** — functions assume string input but don't verify
3. **No type checking at boundaries** — TypeScript helps, but Baileys types are... aspirational
4. **Intermittent conditions** — the wrong shape only appears under specific protocol states

## The AI Agent Twist

In a normal web app, `[object Object]` shows up on screen and someone files a bug. In an AI agent system, the LLM *processes* the garbage and produces a plausible-sounding response. The user gets a wrong answer instead of an obvious error. The feedback loop is broken.

That's the recurring theme in this series: AI systems turn hard failures into soft failures. A crash is obvious. `[object Object]` rendered on a page is obvious. But an LLM confidently responding to `[object Object]` as if it were a real question? That's invisible.

## Takeaways

1. **Guard your boundaries.** Every function that accepts external input should validate type, not assume it. Especially in JavaScript.

2. **`typeof` before `.normalize()`.** If you're calling string methods, prove it's a string first. This applies to `.trim()`, `.split()`, `.replace()` — all of them.

3. **Log the raw input.** If the sanitization function logged `typeof message` on entry, this bug would be caught in minutes instead of weeks.

4. **Don't trust the protocol layer.** Baileys, like many protocol libraries, doesn't guarantee consistent output shapes across all conditions. Treat its output as untrusted.

---

*This is the twelfth post in my "silent failures in AI agents" series. Previous entries: [streaming tool calls](/posts/when-your-agents-tool-call-vanishes-mid-stream/), [WhatsApp append messages](/posts/the-message-that-got-blue-ticks-but-never-arrived/), [lane lock deadlocks](/posts/the-deadlock-that-killed-your-agents-session/).*

*Issue: [openclaw/openclaw#52464](https://github.com/openclaw/openclaw/issues/52464)*
