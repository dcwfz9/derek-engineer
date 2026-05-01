# derek.engineer

Personal project blog. Built with Hugo + PaperMod, hosted on Netlify.

## Stack

- **Generator:** Hugo 0.161.1 with PaperMod theme
- **Hosting:** Netlify (auto-deploys on push to `main`)
- **Domain:** derek.engineer via Squarespace DNS

## Local dev

```bash
hugo server --buildDrafts
# → http://localhost:1313
```

## Adding a post

Create a new file in `content/posts/`:

```bash
hugo new content posts/my-post-title.md
```

Or just create the file manually. Front matter template:

```markdown
---
title: "Post Title"
date: 2026-05-01
draft: false
tags: ["tag1", "tag2"]
description: "One line summary shown in post list."
---

Post content here.
```

Set `draft: false` when ready to publish. Push to `main` — Netlify builds and deploys automatically.

## DNS setup (Squarespace)

| Type  | Name          | Data                              |
|-------|---------------|-----------------------------------|
| A     | derek.engineer | 75.2.60.5                        |
| CNAME | www           | wonderful-fox-6a6371.netlify.app  |

## Netlify

- Site: `wonderful-fox-6a6371.netlify.app`
- Build command: `hugo`
- Publish directory: `public`
- Hugo version pinned in `netlify.toml`
