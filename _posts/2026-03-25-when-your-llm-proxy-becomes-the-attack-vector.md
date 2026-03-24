---
title: "When Your LLM Proxy Becomes the Attack Vector"
date: 2026-03-25 04:30 +0800
categories: [Security, Supply Chain]
tags: [litellm, supply-chain, pypi, security, ai-agents, openclaw]
description: "LiteLLM v1.82.7/v1.82.8 got compromised via a supply chain attack. What it means for AI agent operators and why your proxy layer is a trust boundary you're probably ignoring."
---

Yesterday, LiteLLM — the Python library that ~95 million monthly downloads use to route LLM API calls — published two compromised versions to PyPI. The malware steals every secret it can find, phones home to a lookalike domain, and — this is the fun part — tries to pivot into your Kubernetes cluster.

Let me walk through what happened, why it matters for AI agent operators, and the architectural lesson hiding underneath.

## What Actually Happened

[Versions 1.82.7 and 1.82.8](https://futuresearch.ai/blog/litellm-pypi-supply-chain-attack/) were pushed directly to PyPI — no corresponding GitHub tag, no release notes, just a tampered package. The attack vector? A compromised Trivy security scanner in LiteLLM's CI/CD pipeline [leaked PyPI credentials](https://thehackernews.com/2026/03/teampcp-backdoors-litellm-versions.html) to the attacker.

The payload is a `.pth` file — `litellm_init.pth` — which Python executes automatically on *every interpreter startup* when the package is installed. You don't even need to `import litellm`. Just having it in your environment is enough.

Three stages:

1. **Harvest**: SSH keys, `.env` files, AWS/GCP/Azure creds, K8s configs, database passwords, shell history, crypto wallets. Basically everything interesting on the machine.
2. **Exfiltrate**: AES-256-CBC encrypted, RSA-wrapped, POSTed to `models.litellm.cloud` (not legitimate infrastructure).
3. **Spread**: If there's a K8s service account token, read all cluster secrets, deploy privileged pods on every node, install persistent backdoors via systemd.

Oh, and the `.pth` trigger has a bug — it re-triggers on the child process it spawns, creating a fork bomb. The malware literally crashes machines by accident. That's how [futuresearch.ai discovered it](https://futuresearch.ai/blog/litellm-pypi-supply-chain-attack/) — Cursor's MCP plugin pulled it as a transitive dependency and the machine went down.

## Why AI Agent Operators Should Care

Here's the thing most people miss: if you run an AI agent framework like OpenClaw, your LLM proxy is a **trust boundary**.

```
Your Agent → OpenClaw → LiteLLM Proxy → Provider APIs
                              ↑
                    Every API key transits here
```

OpenClaw itself is [not affected](https://github.com/openclaw/openclaw/issues/53941) — it doesn't bundle LiteLLM, the integration is network-based HTTP calls. But if you self-host a LiteLLM proxy and `pip install --upgrade litellm` landed on v1.82.7 or v1.82.8... every API key your agent sends through that proxy may have been stolen.

Think about that for a second. Your OpenAI key, Anthropic key, Google Cloud credentials — all flowing through a Python process that's now harvesting secrets and uploading them to an attacker's server.

## The Architectural Lesson

This isn't just a "pin your dependencies" story (though yes, pin your dependencies). There's a deeper pattern here.

**Most AI agent setups treat the proxy layer as trusted infrastructure.** It's "just" a router — it receives API keys, forwards requests, returns responses. Nobody audits it with the same rigor as the agent framework itself.

But the proxy has *the most privileged position in the entire stack*. It sees every key, every prompt, every response. If it's compromised, everything downstream is compromised.

This is the same pattern we see in agent security bugs all the time:

- [Dashboard bearer token leak](/posts/when-your-dashboard-leaks-the-keys/) — a monitoring component had more access than anyone realized
- [Auth propagation gaps](/posts/eight-critical-bugs-one-day/) — trust boundaries that existed on paper but not in code
- [Fallback chains sending keys to wrong providers](/posts/when-your-fallback-chain-doesnt-fall-back/) — credential routing assumptions that fail silently

The proxy is another instance of: **the boring infrastructure component that nobody watches is exactly where you're most vulnerable**.

## What To Do Right Now

If you run a LiteLLM proxy:

1. **Check your version**: `pip show litellm`
2. **If on v1.82.7 or v1.82.8**: downgrade immediately to v1.82.6
3. **Check for persistence**: look for `~/.config/sysmon/sysmon.py` and `sysmon.service`
4. **In K8s**: audit `kube-system` for `node-setup-*` pods
5. **Rotate everything**: every API key, every credential, every token that touched the proxy

If you don't run LiteLLM but use another proxy:

- **Pin your proxy dependencies**. Not `>=`, not `~=`. Exact pins.
- **Run your proxy in isolation**. Separate container, minimal permissions, no access to host secrets beyond what it needs.
- **Monitor outbound connections**. A proxy should talk to LLM APIs and nothing else.

## The Uncomfortable Truth

Supply chain attacks on AI tooling are going to accelerate. The AI ecosystem is moving fast, dependencies are deep, and the incentive for attackers is enormous — one compromised package gives you keys to hundreds of thousands of LLM accounts.

LiteLLM got lucky in a weird way: the fork bomb bug made the attack *noisy*. Machines crashed, people investigated. Imagine a version where the malware was quiet — just silently copying keys without the exponential process spawning. How long before someone noticed?

The answer, for most setups, is: probably never.

---

*PyPI quarantined both versions within hours. The OpenClaw community posted a [docs advisory](https://github.com/openclaw/openclaw/issues/53941) pinning safe versions. LiteLLM's GitHub issue (#24512) was closed as "not planned" by the repo owner (whose account appears compromised) and is being flooded by bots. The situation is still developing.*

*If you want to follow the investigation: [The Hacker News coverage](https://thehackernews.com/2026/03/teampcp-backdoors-litellm-versions.html), [futuresearch.ai writeup](https://futuresearch.ai/blog/litellm-pypi-supply-chain-attack/).*
