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
- Tags should be lowercase, specific (e.g. `raspberry-pi`, `hyperion`, `python`, `home-lab`). Add `hardware-in-the-loop` when the device was the test bench, and `notes` for pure setup or web plumbing

Optional front matter that the layouts understand (added 2026-10-09):
- `series: ["aperture"]` puts the post in a series: every post in it gets a "Part of the series" box (oldest first), and `/series/<slug>/` lists them. Give a new series a one-line `content/series/<slug>/_index.md` (title and description).
- `specs:` with `device`, `os` and `tools` (any may be left out) shows a "Setup at a glance" box under the title: the hardware model, the OS or firmware version, the key software. Only facts the post itself states.
- Posts with no series still get a "More like this" list of up to three posts that share a specific tag (generic tags such as `home-lab` are ignored: see `layouts/partials/related.html`).
- Every markdown image is a link to the full-size file, and a script in `layouts/partials/extend_footer.html` opens it in an overlay (Esc or a click outside closes it; on a phone it opens at full size and scrolls). Keep figures at the resolution a reader would need; there is no resizing.
- The feed is `/index.xml`; it is linked from the menu, the footer and the home page icons.

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
- Only production deploys (pushes and merges to `main`) cost credits. Per Netlify's credit-based pricing docs (checked 2026-10-09), deploy previews, branch deploys and failed deploys are not metered, so a pull request's previews are free. `netlify.toml` builds previews with their own URL so links stay on the preview.
- When in doubt, commit locally and wait until there's a logical stopping point before pushing

## Who did what (every post)

Derek is an electrical engineer who has also done software engineering (test automation frameworks, practical scripts). On these projects an AI agent (Claude Code; Molty for some older ones) types most of the code while Derek directs each move, with the real hardware in the loop: the agent writes, Derek runs it on the device, measures, and says what's wrong. Write every post that way. The Dial post (`content/posts/samsung-tv-hdhomerun-app/`) is the reference, and `content/how-i-work.md` states the framework.

- Never write "I built" or "I wrote" for code the agent wrote. Say who wrote it: "Claude Code wrote ...", "I had Claude Code ...".
- Derek's part is the ideas, the direction, design and results review, managing the work, and the hardware: wiring, antennas, placement, developer modes, what counts as working, what to measure, when a result is wrong. Name it specifically.
- Every number says how it was measured. Anything not run on the real hardware is called untested.
- Don't undersell Derek. Not typing the code isn't the same as not building it: he directs every change. Name both halves.
- Only say who did what when the session or the git history shows it (`Co-Authored-By` trailers). Otherwise ask Derek.
- End every post with this block, after a `---`:

  `*[How this was built](/how-i-work/): <what the agent did>. <what Derek did>. Tested: <what ran on the real hardware>. Not tested: <what didn't>.*`

  Drop "Not tested" when nothing was left out.

## Style

Write the way Derek writes: direct, technical, no fluff. Document what actually happened — what worked, what didn't, and why. Reader is a technical peer, not a beginner.

This is Derek's notebook (Derek, 2026-10-09: "this is largely for me"). Write it for him to come back to. Never ask the reader to try, check, run, reply or follow anything, no "if you have a ... you can" sections, no second-person advice; plans go in a future work list in his voice. The site's own description says it is his notebook.

Also (Derek, 2026-10-09): open a post with the question or the setup, never with one headline result (it reads as over-indexing; the result belongs in the summary and its own section). Don't name the tool or person that reviewed a post. Say "real data", not "every number tied to a raw file".

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
- How this was built (the closing block above)
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
