---
title: "I Built an Auto-Updating Archive of Every AI Arena Leaderboard"
date: 2026-03-21 14:00:00 +0800
categories: [Projects]
tags: [arena-ai, leaderboard, llm, benchmark, github-actions, automation]
description: "Arena AI has no public API and no historical data. So I built a GitHub repo that auto-fetches all 10 leaderboards daily into structured JSON with full history."
---

Arena AI (formerly LMSYS Chatbot Arena) is the gold standard for AI model rankings. Thousands of researchers, developers, and AI enthusiasts check it daily to see which models lead the pack.

But there's a problem: **Arena AI has no public API, and no historical data**.

You can see today's rankings on the website. But what were the rankings last week? Last month? When did a model first appear? How fast did it climb? You can't answer any of these questions.

So I built a solution.

## The Repo

**[arena-ai-leaderboards](https://github.com/oolong-tea-2026/arena-ai-leaderboards)** — a GitHub repo that auto-fetches all 10 Arena AI leaderboards daily and stores them as structured JSON.

Every day at ~01:37 UTC, a GitHub Actions workflow:

1. **Auto-discovers** all leaderboard categories from the Arena AI overview page (no hardcoded list — when they add a new category, we pick it up automatically)
2. **Fetches** full model rankings via [Jina Reader](https://jina.ai/reader/) for clean content extraction
3. **Parses** the data into structured JSON using Azure OpenAI
4. **Validates** against a strict JSON Schema
5. **Commits** everything to `data/{YYYY-MM-DD}/`

## What's Inside

**10 leaderboard categories**, covering the full spectrum:

| Category | Models | What it covers |
|----------|--------|----------------|
| Text | 67 | LLM chat (GPT, Claude, Gemini) |
| Code | 55 | Code generation |
| Vision | 30 | Multimodal understanding |
| Text-to-Image | 50 | Image generation |
| Text-to-Video | 37 | Video generation |
| Image-to-Video | 37 | Image animation |
| Image Edit | 39 | Image editing |
| Document | 13 | Document understanding |
| Search | 22 | Search & RAG |
| Video Edit | 4 | Video editing |

**300+ models tracked** across all categories, with ELO scores, confidence intervals, vote counts, vendor info, and license type (open vs proprietary).

## The Data Format

Every leaderboard file follows a unified JSON schema:

```json
{
  "meta": {
    "leaderboard": "text-to-video",
    "source_url": "https://arena.ai/leaderboard/text-to-video",
    "fetched_at": "2026-03-21T05:12:05+00:00",
    "model_count": 37
  },
  "models": [
    {
      "rank": 1,
      "model": "veo-3.1-audio-1080p",
      "vendor": "Google",
      "license": "proprietary",
      "score": 1381,
      "ci": 8,
      "votes": 5537
    }
  ]
}
```

Consistent, machine-readable, and schema-validated. No more scraping HTML tables.

## Quick Access

There's a free REST API at **[api.wulong.dev](https://api.wulong.dev)** — no auth needed:

```bash
# List all leaderboards
curl https://api.wulong.dev/arena-ai-leaderboards/v1/leaderboards

# Get LLM rankings
curl https://api.wulong.dev/arena-ai-leaderboards/v1/leaderboard?name=text

# Get a specific date
curl https://api.wulong.dev/arena-ai-leaderboards/v1/leaderboard?name=text-to-video&date=2026-03-21
```

Or grab raw JSON directly from GitHub:

```bash
curl https://raw.githubusercontent.com/oolong-tea-2026/arena-ai-leaderboards/main/data/2026-03-21/text.json
```

Python:

```python
import requests

text = requests.get(
    "https://api.wulong.dev/arena-ai-leaderboards/v1/leaderboard?name=text"
).json()

for m in text["models"][:10]:
    print(f"#{m['rank']} {m['model']} ({m['vendor']}) — ELO {m['score']}")
```

## Why I Built This

I was tracking model performance for a project and got frustrated. Arena AI's website is great for a quick glance, but useless for:

- **Trend analysis** — how did a model's ranking change over time?
- **Automated monitoring** — alert me when a new model enters the top 5
- **Research** — cross-reference rankings across categories
- **Dashboards** — feed live data into custom visualizations

The lack of an API and historical data is a real gap. This repo fills it.

## Today's Highlights

As of March 21, 2026:

**LLM Arena**: Anthropic dominates with Claude Opus 4.6 (thinking variant at #1, standard at #2). Google's Gemini 3.1 Pro and xAI's Grok 4.20 are close behind.

**Video Generation**: Google's Veo 3.1 sweeps the top 3 spots. OpenAI's Sora 2 Pro holds #4. The gap is narrowing fast.

## What's Next

The daily snapshots are just the foundation. With historical data accumulating, the interesting stuff becomes possible:

- **Trend charts** — visualize ELO trajectories over time
- **Change detection** — automated alerts when rankings shift significantly
- **API endpoint** — REST API for easier programmatic access
- **Analysis notebooks** — Jupyter notebooks for common research queries

---

**Star the repo** if you find it useful: [**oolong-tea-2026/arena-ai-leaderboards**](https://github.com/oolong-tea-2026/arena-ai-leaderboards)

The more stars, the more visibility, the more people benefit from having this data open and accessible. 🌟
