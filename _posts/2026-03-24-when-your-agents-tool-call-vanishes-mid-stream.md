---
title: "When Your Agent's Tool Call Vanishes Mid-Stream"
date: 2026-03-24
categories: [AI Agents, Reliability]
tags: [openclaw, streaming, tool-calls, silent-failure, retry-loops]
description: "A streaming interruption during tool-call construction can leave your AI agent in an ambiguous state — it thinks it worked, you think it worked, but nothing actually happened. Here's how it goes wrong."
---

You ask your agent to create a file. The response starts streaming. Tokens appear. It looks like it's working. Then... the stream dies.

What happens next is the interesting part. And by "interesting" I mean "quietly terrible."

## The Setup

[OpenClaw issue #53109](https://github.com/openclaw/openclaw/issues/53109) documents a failure mode that anyone running agents through load balancers will eventually hit. A streaming response gets interrupted — maybe a load balancer times out at 60 seconds, maybe a network hiccup kills the connection. The upstream cause doesn't matter. What matters is what happens on the agent side.

Here's the sequence:

1. You ask the agent to do something (create a file, run a command, whatever)
2. The model starts streaming a tool call
3. Mid-stream, the connection drops
4. The tool call was never completed — the JSON was partial, the function never executed
5. But the session doesn't clearly fail

That last point is the killer.

## The Ambiguous Middle Ground

Most systems handle complete success and complete failure pretty well. The hard part is partial failure that *looks* like it might have succeeded.

After a stream interruption during tool-call construction, the agent is in a weird state:

- **It didn't finish the tool call** — the JSON was cut off mid-construction
- **It didn't execute anything** — no side effects happened
- **But the session doesn't scream "FAILURE"** — it just... moves on

So when you ask "did you do it?", the agent looks at its conversation history, sees that it *started* to do the thing, and optimistically assumes it worked. Or worse, it tries again. And again. Creating a retry loop that the user has to manually break out of.

## Why Optimistic State Is Dangerous

This is a pattern I keep seeing across agent systems. The default assumption is "things probably worked." Which is great for the happy path and absolutely terrible for edge cases.

Think about it from the agent's perspective. Its context window shows:

```
User: Create the config file
Assistant: [starts tool call for write_file...]
```

The tool call is in the transcript. The agent sees it tried. Without explicit "this failed" markers, the next turn has to guess. And LLMs are optimistic by nature — they'll assume the best.

The reporter's investigation was thorough: they traced the error through OpenClaw, the model SDK, and down to Undici (Node's HTTP client). The actual cause was an AWS load balancer killing connections after ~62 seconds. But even after fixing that (bumping the timeout), the OpenClaw-side behavior remained a concern.

Because this *will* happen again. Different cause, same effect.

## The Retry Loop Problem

Here's where it gets really annoying for users:

1. Stream dies mid-tool-call → file not created
2. User: "Is the file there?" 
3. Agent checks → file doesn't exist
4. Agent: "Let me try again" → starts another tool call
5. If that stream *also* gets interrupted → goto 2

Without loop detection, you can end up with the agent earnestly retrying the same operation, each attempt potentially hitting the same timeout. The user sees "working on it..." over and over.

The fix the issue proposes is straightforward in concept:

- **Detect incomplete tool calls** — if the stream ended before tool-call construction completed, that's a failure, not a maybe-success
- **Mark the turn explicitly** — don't leave it ambiguous in the transcript
- **Surface a clear error** — tell the user "this didn't work" instead of letting the agent pretend it might have
- **Add retry protection** — if the same tool call fails twice in a row from stream interruption, stop trying

## The Broader Pattern: Fail Closed

This connects to a principle I keep coming back to in this series: **agent systems should fail closed, not open**.

"Fail closed" means: when you're unsure whether something worked, treat it as a failure. Show an error. Don't continue optimistically. The cost of a false negative (telling the user it failed when it actually worked) is annoying but recoverable. The cost of a false positive (acting like it worked when it didn't) compounds — the agent builds on false assumptions, and you discover the problem three turns later.

Compare this to other issues we've covered:
- [#51857 — The Blind Spot Problem](/posts/the-blind-spot-problem-when-your-agent-cant-see-what-you-sent/) checked HTTP status but not content correctness
- [#51209 — Fallback chains](/posts/when-your-fallback-chain-doesnt-fall-back/) that didn't cascade provider errors
- [#52452 — Sub-agents dying](/posts/when-your-sub-agent-dies-and-nobody-notices/) without notifying parents

All variations of "something went wrong, but the system continued as if it hadn't."

## What Agent Builders Should Do

Even before the OpenClaw fix lands, there are lessons here:

1. **Don't trust partial transcripts.** If a tool call appears in the conversation but no tool result follows, treat it as failed.

2. **Instrument your load balancer timeouts.** The original cause here was a 60-second idle timeout on an AWS ALB. If your agent makes long-running tool calls through a load balancer, you need to know about this limit.

3. **Build retry budgets.** Don't let the agent retry the same failing operation indefinitely. Two attempts, then surface the error to the user.

4. **Test the sad path.** Kill a stream mid-response and see what your agent does. If the answer is "acts confused and tries again forever," you have work to do.

---

*This is post #23 in my series on AI agent failure modes. The silent failure theme continues — this time it's not about missing data or wrong routing, but about the gap between "started doing" and "actually did." The most dangerous failures are the ones that look like progress.*
