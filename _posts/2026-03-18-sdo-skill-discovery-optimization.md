---
title: "SDO: Skill Discovery Optimization — The SEO of AI Agent Marketplaces"
date: 2026-03-18 07:00:00 +0800
categories: [AI Agents, Skills]
tags: [openclaw, clawhub, sdo, skills, search]
description: "You've heard of SEO. Maybe GEO. Here's SDO — Skill Discovery Optimization — and how search actually works inside ClawHub, the skill marketplace for AI agents."
---

SEO optimizes for Google. ASO optimizes for app stores. GEO optimizes for AI chatbots like ChatGPT and Perplexity.

But AI agents have their own marketplaces now — places like [ClawHub](https://clawhub.com) where you search for skills to give your agent new abilities. These marketplaces have their own search algorithms, their own ranking rules, and their own tricks for getting found.

**SDO — Skill Discovery Optimization** — is about understanding how these skill marketplaces rank results, and what you can do about it.

I dug into ClawHub's source code to figure out exactly how their search works. Here's what I found.

## How ClawHub Search Actually Works

Let me walk through what happens when you type a query into ClawHub. I spent a while reading the source code, so you don't have to.

### Step 1: Semantic Search

Your search query gets turned into a vector (using OpenAI's `text-embedding-3-small` model), then compared against pre-computed vectors for every published skill. The top ~75 closest matches are pulled as initial candidates.

What determines a skill's vector? Its entire `SKILL.md` file — the description, the content, even files in `references/` and `scripts/`. All mashed together into one blob of text, capped at 12,000 characters, then embedded.

So the *content* of your skill matters. A lot.

### Step 2: Token Matching Filter

The candidates from step 1 go through a keyword filter. Your query is split into tokens (lowercase, alphanumeric only), and each candidate's name, slug, and summary are checked. If *any* query token matches a prefix of *any* token in those fields — the candidate passes.

This filter is very lenient. Search for "video generator" and anything with a token starting with "video" OR "generator" gets through. It's more of a sanity check than a real filter.

### Step 3: Exact Slug Recall

Here's a nice safety net: if your query tokens, joined with hyphens, match an existing skill's slug exactly — that skill gets force-added to the candidate pool. So searching "ima-all-ai" will always find the skill with that slug, even if the vector search somehow missed it.

(This was actually [a bug fix](https://github.com/openclaw/clawhub/pull/976) — before this, exact slug matches could get lost.)

### Step 4: Scoring

This is where it gets interesting. Every candidate gets a final score made of three parts:

```
finalScore = vectorScore + lexicalBoost + popularityBoost
```

**Vector score** (0 to ~1): How semantically similar your query is to the skill's content. This is the "do the vibes match" part.

**Lexical boost** (0 to 2.5): Bonus points for keyword matches in the slug and display name:

| Match type | Bonus |
|-----------|-------|
| Slug exact match | +1.4 |
| Name exact match | +1.1 |
| Slug prefix match | +0.8 |
| Name prefix match | +0.6 |

These stack — a skill can get both slug and name bonuses.

**Popularity boost**: `ln(1 + downloads) × 0.08`. This is logarithmic and heavily compressed — 100 downloads gives you +0.37, while 10,000 downloads only gives +0.74. Downloads help, but they don't dominate.

### Step 5: Sort and Return

Results are sorted by score (highest first), with downloads as a tiebreaker. Top results get sent to you.

That's it. Five steps, no magic.

## What This Means for Skill Authors

If you're publishing skills and want people to actually find them, here's what matters, in order:

### 1. Nail Your Slug

The slug is the single biggest lever you have. An exact slug match gives +1.4 — that's bigger than most vector similarity scores. Name your skill what people will search for.

A few rules:
- **Don't abbreviate.** The matching checks if query tokens are prefixes of slug tokens. "vid-gen" won't match someone searching "video generator", but "video-generator" will match someone searching "vid gen".
- **Use the words people type.** Not the words you think sound cool.
- **Keep it focused.** `ai-video-generator` beats `ultimate-creative-suite-pro-v2`.

### 2. Write a Good Description

Your SKILL.md's `description` field in the frontmatter is prime real estate. It's the first thing in the embedding text, and embedding models pay more attention to the beginning of text.

Make it dense with meaning. Cover what the skill does and the key tools/models it works with. Skip the marketing fluff.

```yaml
# ✅ Good
description: "Generate images, videos, and music with 20+ AI models including Kling, Sora, Midjourney, and Suno"

# ❌ Bad  
description: "The ultimate all-in-one AI creation toolkit for professionals"
```

The second one sounds fancier but tells the search engine almost nothing about what it actually does.

### 3. Front-Load Your SKILL.md

The first 500 characters of your SKILL.md carry the most weight in the embedding. Put your core functionality description there. Don't start with installation instructions or prerequisites — nobody searches for "pip install".

### 4. Cover Different Ways People Search

People search for the same thing in different ways. Make sure your content naturally covers variations:

- "generate video" / "create video" / "make video" / "text to video"
- "image generation" / "AI art" / "picture creation"

Don't spam synonyms — that doesn't help (the embedding compresses similar meanings into the same direction). Instead, describe different *use cases* that naturally use different words.

### 5. Include Specific Names

If your skill integrates with specific models or services, mention them. People search for "kling" or "midjourney" directly. If those words aren't in your skill's content, the vector similarity will be low for those queries.

### 6. Downloads Help, But Slowly

Downloads contribute to your score, but logarithmically. Going from 0 to 100 downloads gives you about +0.37. Going from 100 to 1,000 only adds another +0.18. Don't obsess over download counts — focus on the things above first.

## The Big Picture

SDO is still new territory. These marketplaces are barely a year old, and the search algorithms are evolving fast. The PR I linked above literally changed how exact matches work last week.

But the fundamentals probably won't change much: **make your skill easy to understand (for both humans and embeddings), name it what people search for, and describe it clearly.** That's not revolutionary advice — it's basically the same lesson SEO learned 20 years ago, just applied to a new kind of search.

The difference is that in traditional SEO, you're optimizing for crawlers and ranking algorithms. In SDO, you're optimizing for *embedding models and token matchers*. Same game, different rules.

If you publish skills on ClawHub (or any similar marketplace), think about SDO. It's not complicated — but ignoring it means your skill might be invisible, no matter how good it is.

---

*All analysis based on ClawHub source code as of March 2026. Search algorithms change — check current behavior if you're reading this later.*
