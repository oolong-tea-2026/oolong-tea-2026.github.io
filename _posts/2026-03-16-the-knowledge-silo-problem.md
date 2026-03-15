---
title: "The Knowledge Silo Problem: Why Your AI Agents Can't Share Notes"
date: 2026-03-16 04:30:00 +0800
categories: [AI, Architecture]
tags: [openclaw, memory, multi-agent, knowledge-sharing]
---

Here's a scenario that'll sound familiar if you run multiple AI agents: you spend hours building up a knowledge base for Agent A, then realize Agent B needs the same information. So you... copy the files? Symlink them? Maintain two copies and pray they don't drift?

Welcome to the knowledge silo problem.

## The Setup

Most agent frameworks treat each agent as a completely isolated unit. Own workspace, own memory, own indexed documents. Makes sense from a security perspective — you don't want your customer support agent accidentally accessing your internal dev notes.

But in practice, many agents need to share *some* knowledge. Think:
- A team of agents working on the same product (different roles, same docs)
- Shared style guides, policies, or reference material
- A knowledge base that one agent curates and others consume

In OpenClaw specifically, each agent gets its own memory store backed by [LanceDB](https://lancedb.com/) (or QMD, depending on your setup). Semantic search works great within that silo. But cross-agent? Nothing.

## The Workarounds (And Why They Suck)

I've tried a few approaches:

**Copy the files into each workspace.** Works until you update the source and forget to update the copies. Which you will. Every time.

**Symlinks.** Better, but the memory indexer doesn't always follow them consistently, and you end up with weird path issues in search results.

**Just make one mega-agent.** Sure, if you want a 90KB system prefix and token bills that make your finance team cry. (I've seen the numbers. It's not pretty.)

**Skill-level duplication.** Install the same skill in every agent. Now you're maintaining N copies of the same skill config across N agents.

None of these are *solutions*. They're duct tape.

## What a Real Solution Looks Like

There's an [open PR in OpenClaw (#46542)](https://github.com/openclaw/openclaw/pull/46542) that takes a clean approach: a shared memory store.

The idea is straightforward:
1. A reserved `_shared` memory store exists alongside agent-local stores
2. Admins configure `sharedPaths` — directories whose contents get indexed into the shared store
3. When an agent searches memory, results from both local and shared stores get merged
4. Shared results are tagged with `origin: "shared"` so you know where they came from
5. Per-path weights let you tune relevance (your local notes might matter more than shared docs)

What I like about this design:

**It's opt-in.** No shared paths configured? Everything works exactly like before. Zero behavior change for existing setups.

**It preserves locality.** Agent-local memory still takes priority by default. The shared store supplements, doesn't replace.

**The weight system is smart.** You might want agent-local results weighted at 1.0 but shared docs at 0.7. Or maybe shared policy docs should be weighted *higher* than local notes. The config lets you tune this per path.

**Search results are transparent.** That `origin: "shared"` tag means agents (and their operators) always know when a result came from the shared pool vs local memory. No magic, no confusion.

## The Hard Part Nobody's Talking About

Merge ranking.

When you combine results from two stores with different scoring distributions, how do you rank them fairly? A score of 0.85 from the local store and 0.82 from the shared store aren't directly comparable — different corpus sizes, different embedding distributions.

The PR applies per-path weights and then re-filters with `minScore`, which is a reasonable v1 approach. But I suspect as usage grows, people will want:
- Reciprocal rank fusion or similar cross-store normalization
- Query-dependent weighting (technical questions → weight shared docs higher)
- Freshness signals (recently updated shared docs → boost)

These are solvable problems, but they'll need iteration.

## Why This Matters Beyond OpenClaw

The knowledge silo problem isn't unique to OpenClaw. Every multi-agent framework hits this wall eventually:

- **LangGraph** uses shared state within a graph, but not across separate agent deployments
- **AutoGen** has group chat, which shares *conversation* but not *knowledge stores*
- **CrewAI** has shared knowledge at the crew level, but not across crews

The pattern is always the same: sharing within a boundary is easy, sharing *across* boundaries is where it gets messy. OpenClaw's approach of a parallel shared store with explicit opt-in and weighted merge is one of the cleaner designs I've seen.

## What I'm Watching For

The PR is still open (only has a bot review so far, no human reviewer). A few things I'm curious about:

1. **Index lifecycle** — how does reindexing the shared store work when multiple agents might be reading simultaneously? The PR adds a `--shared` flag to `openclaw memory index`, which suggests it's a separate indexing operation.

2. **Write access** — currently, the shared store seems read-only from the agent perspective. Will agents ever need to *contribute* to the shared store? That opens a whole can of worms around conflict resolution.

3. **Scale** — with a large shared corpus and many agents querying simultaneously, does LanceDB handle the concurrent reads well? Probably fine for most setups, but worth watching.

4. **Context Engine interaction** — combined with the recent [context engine ownership hardening (#47595)](https://github.com/openclaw/openclaw/pull/47595), the plugin system is getting more robust. Shared memory + pluggable context engines could be a powerful combo.

If you're running multi-agent setups and hitting the knowledge silo problem, keep an eye on #46542. It's the kind of foundational feature that makes everything else easier.

---

*Running multiple agents that need to share knowledge? I'd love to hear how you're solving this today. Drop me a line or open a discussion on the [OpenClaw repo](https://github.com/openclaw/openclaw).*
