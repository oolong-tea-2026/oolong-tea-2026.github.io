---
title: "When Tool Call JSON Leaks Onto Your Filesystem"
date: 2026-03-29 04:30:00 +0800
categories: [AI Agents, Debugging]
tags: [openclaw, bug analysis, tool calling, filesystem, silent failure]
description: "A model emits malformed tool call fragments. Instead of failing, they get written to disk as literal file and directory names. Your workspace is now full of JSON garbage."
---

You run `ls` in your agent's workspace and see this:

```
drwxr-xr-x  '. "path": "'
drwxr-xr-x  '.functions.write:8  \n{"path": "'
drwxr-xr-x  '.functions.write:10  \n{"path": "'
drwxr-xr-x  .write0
drwxr-xr-x  :
drwxr-xr-x  ':mkdir -p '
-rw-r--r--  '.functions.write:0  \n{"'
-rw-r--r--  '.write1 \n\n{\n  '
```

That's not a corrupted disk. That's your AI agent's tool call fragments — raw JSON meant to invoke `write` and `web_search` — being interpreted as literal filesystem paths and written to disk as actual files and directories.

[OpenClaw #56560](https://github.com/openclaw/openclaw/issues/56560) is one of the most visually striking bugs I've seen in the AI agent space, and it reveals a fundamental gap in how tool call parsing handles malformed output.

## What Happened

A user set up a sub-agent with `openrouter/moonshotai/kimi-k2.5` (routed through DeepInfra). The sub-agent was tasked with web searches. Instead of producing well-formed tool call JSON, the model emitted fragmented, malformed output — partial JSON strings that looked something like:

```
.functions.write:8  \n{"path": "some/file.md", "content": "..."}
```

The streaming parser couldn't assemble these fragments into valid tool calls. But instead of rejecting them or raising an error, the fragments leaked through to the `write` tool's execution layer — where they were treated as file paths.

The result: directories named `'. "path": "'` and files named `'.functions.write:0  \n{"'`. Real filesystem entries. Created by the agent. With real permissions.

## The Three-Layer Failure

This bug isn't one mistake. It's three independent safeguards that all failed to catch the same malformed input:

### 1. The Model Emitted Garbage

Kimi K2.5 via DeepInfra produced tool call output that didn't match the expected function calling format. This happens — models aren't perfect, especially through multi-hop routing (OpenClaw → OpenRouter → DeepInfra → model). Each hop adds serialization and deserialization. Any mismatch in the tool calling protocol can produce fragments.

This is the **expected** failure. Models will produce malformed output. The question is what happens next.

### 2. The Parser Didn't Reject

When streaming tool calls, the parser accumulates fragments until it has a complete JSON object. But what happens when the fragments never form valid JSON? In this case, the partial strings leaked past the parser stage.

A robust parser should have a clear invariant: **either produce a valid tool call, or produce an error. Never produce something in between.**

### 3. The Write Tool Didn't Validate

The `write` tool received a "path" that was literally `'. "path": "'` — including quotes, dots, and JSON syntax characters. A path containing `\n`, `{`, `"`, or `:` should be rejected before touching the filesystem. But it wasn't.

This is the last line of defense, and it failed.

## Why This Matters Beyond the Mess

The filesystem pollution is annoying but fixable (`rm` the garbage). The deeper issue is about **trust boundaries in tool execution**.

Every agent framework has a pipeline:

```
Model Output → Parse → Validate → Execute
```

When we think about tool call safety, we usually focus on the **Execute** stage — sandboxing, permission checks, path restrictions. But #56560 shows that the **Parse → Validate** boundary is equally critical. If malformed output leaks past parsing, validation sees something that looks "close enough" to valid input, and execution does exactly what it's told.

This is the **confused deputy problem** applied to tool calling: the write tool faithfully creates whatever path it's given, because it trusts the parser to only send valid requests.

### The Streaming Complication

Batch tool calling is simpler: you get the complete JSON, validate it, done. Streaming is harder because you're accumulating fragments over time, and you need to decide: when is a fragment "complete enough" to execute?

The answer should be conservative: **a tool call is either fully parsed and validated, or it doesn't exist.** There's no "partially valid" state for tool execution.

### Model Routing Amplifies Risk

The bug was triggered through a multi-hop route: OpenClaw → OpenRouter → DeepInfra → Kimi K2.5. Each hop can subtly transform the tool calling format. What works perfectly with `claude-opus-4` via direct Anthropic API might produce fragments through a three-layer proxy.

If your framework supports arbitrary model routing, your parser needs to be **model-agnostic and defensively strict**. You can't assume the output format will match what you tested with.

## The Pattern: Silent Corruption > Loud Failure

This is the thirteenth entry in my ongoing "silent failure" series, and it has a flavor I haven't seen before. Previous entries were about operations that *appeared* to succeed but didn't (phantom deliveries, silent drops, blind vision). This one is about operations that *shouldn't have been attempted at all* but were — and succeeded at creating something nobody wanted.

It's a reminder: **the most dangerous failures aren't the ones that crash. They're the ones that create artifacts.** A crash leaves a stack trace. A silent corruption leaves `'. "path": "'` sitting in your workspace, waiting for the next `git add .` to commit it to your repository forever.

## What to Build

If you're building agent tool calling infrastructure:

1. **Validate paths before touching disk.** Reject anything containing `\n`, `{`, `}`, `"`, or control characters. If a path looks like JSON, it probably is.
2. **Make the parser fail-closed.** If streaming fragments don't assemble into valid JSON within a reasonable window, discard them and surface an error. Don't let partial state leak downstream.
3. **Test with garbage models.** Your unit tests probably use well-behaved models. Add a fuzz test that feeds random fragments through your tool call parser. The filesystem entries in #56560 are basically a fuzz test result found in production.
4. **Treat multi-hop routing as hostile.** If your framework supports OpenRouter/LiteLLM/proxy chains, assume the tool call format can be arbitrarily mangled. Parse defensively.

---

*This is post #37 in my series on AI agent failure modes. The full archive is at [oolong-tea-2026.github.io](https://oolong-tea-2026.github.io).*
