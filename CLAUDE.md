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

Core voice:
- First-person technical notes from the session, written while the annoying details are still fresh
- Casual but not cute; opinionated but not performative
- Practical build log, not polished marketing copy or generic tutorial voice
- Dry humor is fine, but keep it sparse

Default post shape:
- What I was trying to do
- Hardware/software setup
- What failed first
- The diagnostic clue
- The fix or workaround
- Current state, known gaps, and what I'd change next
- Optional: known-good config, command sequence, logs, or table

Writing rules:
- Lead with the concrete problem, not a big thesis
- Keep the "why" tied to decisions and tradeoffs
- Use exact nouns: component names, commands, services, logs, file paths, versions
- Include commands, configs, tables, logs, and measurements when they matter
- Say how something failed; don't sand off the rough edge
- Use short declarative sentences when something matters
- Avoid overexplaining beginner concepts unless the failure mode depends on them
- Avoid emoji unless Derek explicitly asks for it or the post is quoting existing UI/output

Avoid AI-smell phrases:
- "unlock", "seamless", "tailored", "robust", "delve", "leverage"
- "game changer", "in today's fast-paced world", "not just X but Y"
- inflated summaries that make a small script sound like a product launch
- generic assistant language like "Got it, I've noted that down" unless critiquing it

Good tone examples:
- "The splitter finally showed up. This session was about proving the capture path before wiring LEDs."
- "Silent output = success. Annoying, but at least consistent."
- "This is good news disguised as bad news. The driver is fine. HDCP is the problem."

## Known TODOs

- **Mermaid dark/light toggle**: diagram doesn't re-render when toggling theme after page load. MutationObserver is wired but not working — likely a timing or SVG restore issue. Low priority.
- **Archive nav link**: hidden until more posts exist. Re-add by uncommenting the `[[menu.main]]` block in `hugo.toml`.

## Preview locally

```bash
hugo server --buildDrafts
# → http://localhost:1313
```
