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

## Style

Write the way Derek writes: direct, technical, no fluff. Document what actually happened — what worked, what didn't, and why. Reader is a technical peer, not a beginner.

## Preview locally

```bash
hugo server --buildDrafts
# → http://localhost:1313
```
