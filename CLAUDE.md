# derek.engineer — Agent Instructions

This is Derek's Hugo project blog at derek.engineer. Posts are written in Markdown, committed to git, and auto-deployed via Netlify.

## Repo location
`~/code/derek-engineer/`

## Creating a post

Posts live in `content/posts/`. File name should be kebab-case and descriptive.

Front matter template:
```markdown
---
title: "Title Here"
date: YYYY-MM-DD
draft: false
tags: ["tag1", "tag2"]
description: "One line summary shown in post list."
---
```

- `draft: false` = published immediately on push
- `draft: true` = built locally only, not deployed
- Use today's date unless told otherwise
- Tags should be lowercase, specific (e.g. `raspberry-pi`, `hyperion`, `python`, `home-lab`)

## Publish workflow

```bash
git add content/posts/<filename>.md
git commit -m "Add post: <title>"
git push
```

Netlify auto-deploys on push to `main`. No further action needed.

## Netlify credit limit — IMPORTANT

**300 credits/month. Each production deploy costs 15 credits = 20 deploys max per month.**

Rules:
- Batch all changes in a session into as few pushes as possible — ideally one push per session
- Never push single small fixes separately
- Drafts (`draft: true`) are safe to commit and push — they don't affect the live site but still consume a deploy credit, so batch them too
- When in doubt, commit locally and wait until there's a logical stopping point before pushing

## Style

Write the way Derek writes: direct, technical, no fluff. Document what actually happened — what worked, what didn't, and why. Reader is a technical peer, not a beginner.

## Preview locally

```bash
hugo server --buildDrafts
# → http://localhost:1313
```
