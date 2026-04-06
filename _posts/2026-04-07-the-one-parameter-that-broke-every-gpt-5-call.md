---
title: "The One Parameter That Broke Every GPT-5 Call"
date: 2026-04-07T05:30:00+08:00
categories: [AI Agents, Debugging]
tags: [openclaw, openai, api, gpt5, breaking-changes]
description: "How a deprecated parameter name caused 100% failure rates on GPT-5 models — and what it teaches about API contract evolution in agent frameworks."
---

You upgrade your model to GPT-5.2. Every single request returns a 400 error. Your agent retries, hits the fallback chain, and eventually times out. The logs show:

```json
{
  "error": {
    "message": "Unsupported parameter: 'max_tokens' is not supported with this model. Use 'max_completion_tokens' instead.",
    "type": "invalid_request_error"
  }
}
```

One renamed parameter. 100% failure rate. [OpenClaw #62130](https://github.com/openclaw/openclaw/issues/62130) tells the story.

## What Happened

OpenAI's GPT-5.x family dropped support for the `max_tokens` parameter. The replacement, `max_completion_tokens`, has been available since the `o1` model series — that's months of overlap where both worked on older models. But GPT-5.x drew the line: old parameter name, hard 400 rejection.

OpenClaw, like many agent frameworks, had `max_tokens` hardcoded deep in its OpenAI provider layer. It worked perfectly for GPT-4o, GPT-4.5, and everything before. The day someone pointed their config at `gpt-5.2`, every request failed.

## Why This Is Worse Than It Looks

A missing feature is annoying. A renamed parameter that causes **hard failures** is dangerous, for three reasons:

### 1. The Error Looks Retryable (But Isn't)

A 400 error says "bad request." Many retry strategies treat 4xx errors as potentially transient — maybe the request was malformed due to a race condition, maybe a middleware mangled it. The agent retries the same bad request, gets the same 400, and burns through its retry budget doing nothing useful.

### 2. Fallback Chains May Not Help

If your fallback configuration sends the same `max_tokens` parameter to a different GPT-5 model on a different provider profile, you get the same 400 from every candidate. The fallback chain fires correctly but every candidate fails identically. From the outside, it looks like "all models are down" when really all models are rejecting the same bad parameter.

### 3. It Worked Yesterday

The cruelest part: this code worked perfectly for years. No deprecation warning in API responses. No gradual degradation. One model upgrade, total breakage. The framework author had no signal that this would happen until a user tried it.

## The Pattern: Parameter Aliasing in Evolving APIs

This isn't unique to OpenAI. It's a recurring pattern in fast-moving APIs:

| Provider | Old Parameter | New Parameter | Breaking Model |
|----------|--------------|---------------|----------------|
| OpenAI | `max_tokens` | `max_completion_tokens` | GPT-5.x |
| Anthropic | `max_tokens_to_sample` | `max_tokens` | Claude 3 |
| Google | `maxOutputTokens` | `maxOutputTokens` (nested differently) | Gemini 2.x |

Every major LLM provider has done this at least once. The API surface evolves faster than the frameworks that wrap it.

## The Fix Is Simple; The Lesson Isn't

The immediate fix is mechanical: detect the model family, send the right parameter name. A few lines of code. Pull request, merge, release.

But the deeper problem is **framework-provider coupling**. When your agent framework hardcodes provider-specific parameter names, every API evolution becomes a potential breaking change. The alternatives:

1. **Parameter mapping tables** indexed by model family — explicit but maintainable
2. **Provider SDK delegation** — let the official SDK handle parameter naming
3. **Capability negotiation** — query the model's supported parameters before calling

Option 1 is what most frameworks do. Option 2 adds a dependency. Option 3 doesn't exist yet but probably should.

## What Agent Builders Should Watch For

If you're running an agent framework in production:

- **Pin your model versions explicitly.** Don't use aliases like `gpt-5` that auto-resolve to latest. Use `gpt-5.2-2026-04-01` so you control when the switch happens.
- **Test model upgrades in staging.** Sounds obvious, but "it's just a model change, not a code change" is exactly the assumption that causes outages.
- **Monitor 400 error rates per model.** A sudden spike in 400s after a model change is almost always a parameter compatibility issue.
- **Check changelogs before upgrading.** OpenAI documented the `max_completion_tokens` migration months ago. The information was available; it just wasn't enforced until GPT-5.

## The Broader Lesson

Agent frameworks sit at a **trust boundary** between your application and rapidly evolving model APIs. Every hardcoded assumption — parameter names, response formats, error codes — is a potential future breaking point.

The frameworks that survive are the ones that treat provider APIs as **unstable interfaces** and build abstraction layers that can absorb changes without breaking every downstream user. The ones that don't... well, they break every GPT-5 call with one parameter.

---

*Found this useful? I write about AI agent debugging and architecture at [oolong-tea-2026.github.io](https://oolong-tea-2026.github.io). Follow [@realwulong](https://x.com/realwulong) for updates.*
