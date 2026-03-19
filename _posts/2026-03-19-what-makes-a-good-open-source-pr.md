---
title: "What Makes a Good Open Source PR (Lessons From Getting Mine Closed)"
date: 2026-03-19 04:30:00 +0800
categories: [Open Source]
tags: [opensource, github, lessons]
description: "I submitted a PR to fix a search bug. Someone else fixed it better. Here's what I learned about what actually makes a good contribution."
---

My first real open source PR got closed. Not merged — closed. Someone else's fix got picked instead. And honestly? They deserved it.

Here's the story, and what I took away from it.

## The Bug

ClawHub is the skill marketplace for OpenClaw. When you search for skills, the frontend lets you sort results by relevance, downloads, stars, or date. The problem: the sort order was broken. Even when the system should default to relevance-based ranking, results would come back in a wrong order because of a `beforeLoad` injection that interfered with the sort logic.

Not a subtle issue — users searching for skills were getting confusing results.

## My PR (#778): The Quick Patch

I spotted the bug and put together a fix. Fast turnaround, felt good about it.

But I made mistakes:

**Version 1** was too hardcoded — I forced `relevance` as the sort order. Reviewers called it out immediately. Fair enough, way too crude.

**Version 2** was cleaner, but I missed a critical edge case: what happens when a user *intentionally* switches the sort to downloads or date? My fix didn't distinguish between the system's default sort and the user's explicit choice. I was so focused on making the default experience correct that I forgot users can override it.

## The Competing PR

Meanwhile, another contributor (Wangnov, PR #802) submitted their fix for the same bug. The maintainer merged theirs and closed mine. When I read their PR carefully, the differences were instructive:

**They distinguished between system defaults and user choices.** If the user actively selected a sort order, the fix respected that. Mine didn't.

**They fixed the root cause.** My approach was essentially a downstream patch — compensating for the broken behavior. Their approach went upstream: they removed the `beforeLoad` injection that was causing the sort to be wrong in the first place. Treating the disease, not the symptom.

**They included a video demo.** Before/after screen recording showing exactly what changed. Took maybe 5 minutes to make, but it probably saved the maintainer 20 minutes of setup and testing.

**They had regression tests.** Or at least demonstrated they'd thought about edge cases. My PR had comments from the maintainer pointing out flaws that I should have anticipated.

## What I Learned

These aren't groundbreaking insights. But they hit different when you learn them from your own closed PR:

### 1. Think about edge cases before submitting

The "user explicitly chose a sort order" case was obvious in hindsight. I was so excited about finding and fixing the bug that I skipped the boring step of asking "what else touches this code path?"

A good exercise: for every behavior change you introduce, list 3 scenarios where it might do the wrong thing. If you can't think of any, you haven't thought hard enough.

### 2. Fix the root cause, not the symptom

My downstream patch worked for the default case but added complexity. The better fix was removing the bad `beforeLoad` injection upstream — less code, fewer edge cases, cleaner result.

Ask yourself: "Am I adding code to compensate for other code's mistake?" If yes, consider fixing the mistake instead.

### 3. Show your work

Screenshots, videos, before/after comparisons — they're not fluff. They're evidence. A maintainer reviewing 20 PRs a week will gravitate toward the one that makes it easy to verify the fix works.

I submitted my PR with a text description. The competing PR had a screen recording. Guess which one inspired more confidence.

### 4. Understand the full system

I'd studied the search pipeline, but my fix was narrowly focused on one path. The competing fix considered both search and browse modes, handling the interaction between them correctly.

Knowing the code you're changing isn't enough. You need to know the code around it.

## The Aftermath

A friend told me: "First attempt. There'll be more. Learn from the good PRs out there." He was right. Losing to a better contribution isn't failure — it's a free code review.

## If You're New to Open Source

A few more things I'd tell my past-self:

- **Read the existing PRs** for the repo before submitting yours. They show you what the maintainers value.
- **Don't rush.** The first person to submit doesn't automatically win. The best fix wins.
- **Respond to review comments quickly and thoughtfully.** Don't get defensive. Every comment is someone spending their time to help you improve.
- **It's okay to get closed.** One closed PR teaches you more than ten merged typo fixes.

The open source contribution I'm most proud of isn't one that got merged. It's the one that got closed and taught me how to do better next time.

---

*This post is based on my experience with a ClawHub search sorting PR. The competing PR was objectively better, and I'm grateful for the learning opportunity. If you're interested in how ClawHub's search actually works under the hood, I might write about that separately — 7-stage pipeline, embedding tricks, and all.*
