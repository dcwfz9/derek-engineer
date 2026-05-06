---
title: "How This Blog Is Built and Published"
date: 2026-05-16
draft: true
tags: ["hugo", "netlify", "claude-code", "meta"]
description: "Hugo, PaperMod, Netlify, DNS, search, and the local writing workflow behind derek.engineer."
---

This is how derek.engineer is wired: static Hugo site, PaperMod theme, Netlify deploys, Squarespace DNS, and a local writing workflow that keeps publishing from becoming its own project.

## The stack

- **Hugo** — static site generator, fast builds, zero frontend dependencies
- **PaperMod** — clean Hugo theme, installed as a git submodule
- **Netlify** — deploys on every push to `main`, custom domain + SSL handled automatically
- **GitHub** — source of truth, all posts are Markdown files in `content/posts/`

The flow is simple:

```text
Write Markdown → commit → push to GitHub → Netlify builds and deploys → live in ~30 seconds
```

## Why Hugo

I wanted a blog, not a project — no build pipeline to maintain, no CMS to update, no dependencies that rot. Markdown files in a git repo, deploy on push. Hugo does exactly that and nothing else. `hugo server --buildDrafts` spins up a live preview in under a second.

PaperMod for the theme — minimal, dark mode by default, readable code blocks. I didn't need a custom design, I needed something I could ignore.

## Netlify setup

The `netlify.toml` pins the Hugo version so the production build matches local:

```toml
[build]
  command = "hugo"
  publish = "public"

[build.environment]
  HUGO_VERSION = "0.161.1"
```

Netlify picks this up automatically. New commit to `main` → deploys. No CI to configure, no Actions YAML, no deploy scripts.

Custom domain setup took a bit more than two minutes — here's the actual detail.

## DNS setup

The registrar is Squarespace. Netlify is just the deployment target. So the job was: tell Squarespace DNS to point traffic to Netlify.

Two hostnames matter: the apex domain (`derek.engineer`) and the subdomain (`www.derek.engineer`). They're handled differently.

**The www record** is straightforward — a CNAME pointing to the Netlify site instance:

| Type  | Name | Value                              |
|-------|------|------------------------------------|
| CNAME | www  | wonderful-fox-6a6371.netlify.app   |

**The apex record** is trickier. Apex domains can't be a standard CNAME at most DNS providers. Netlify offers two options:

- **Preferred**: ALIAS record to `apex-loadbalancer.netlify.com`
- **Fallback**: A record to `75.2.60.5`

Squarespace supports ALIAS, so that's what I used:

| Type  | Name | Value                          |
|-------|------|--------------------------------|
| ALIAS | @    | apex-loadbalancer.netlify.com  |

The `@` means the root domain itself.

One thing to watch: at one point there was also a `derek.engineer → A → 75.2.60.5` record alongside the ALIAS. That's the fallback option — valid but redundant when you have ALIAS working. Remove it so there's one clear source of truth for the apex.

**Final DNS shape:**

```text
@    ALIAS  apex-loadbalancer.netlify.com
www  CNAME  wonderful-fox-6a6371.netlify.app
```

Once DNS propagated, Netlify verified both hostnames and provisioned the TLS certificate automatically. Until that finishes (~a few minutes), the browser will show "Not Secure" — expected, not broken.

## Local writing workflow

The part that makes this setup useful in practice is `CLAUDE.md`.

Claude Code reads `CLAUDE.md` before working in the repo. Mine gives it the rules I would otherwise have to repeat every time:

- Front matter format with required fields
- File naming convention (kebab-case, descriptive)
- The publish workflow (`git add`, `git commit`, `git push`)
- Voice and tone guidelines

The result: when I finish a project, I can dump rough notes from the session, have Dex format them into a post, then review the actual Markdown before publishing.

That is the workflow I want: the source stays plain Markdown, the git history stays normal, and the writing assistant is just part of the local toolchain.

## Enhancements added after launch

A few quick PaperMod features enabled after the initial setup:

**Search** — PaperMod ships with Fuse.js-based fuzzy search. Enabling it just required adding JSON to the Hugo output formats and creating a `search.md` page:

```toml
[outputs]
  home = ["HTML", "RSS", "JSON"]
```

**Archive** — Chronological post list grouped by year/month. One `archive.md` page with `layout: archives`.

**Table of contents** — Auto-generated per post from headings. One line in `hugo.toml`:

```toml
ShowToc = true
```

**Tags in nav** — Already generating `/tags/` pages, just needed a menu entry.

## Netlify credit limits

One gotcha worth knowing: Netlify's free tier gives you **300 credits/month**, and each production deploy costs 15 credits. That's 20 deploys per month.

Every `git push` to `main` triggers a deploy. If you're iterating — pushing small fixes, tweaks, and content changes separately — you'll burn through credits fast. I hit 180/300 in a single session by pushing every change individually.

The fix is simple: batch changes locally and push once per session. Commit as much as you want, just don't push until you're done with a logical chunk of work.

## The gotcha

Netlify's free tier gives 300 credits/month, 15 per deploy — 20 deploys total. I burned 180 in one session pushing every small fix separately. Now I batch and push once per session. Simple fix, but worth knowing before you hit it.

The `CLAUDE.md` file is the other thing worth stealing for any similar setup. It keeps the writing workflow explicit: front matter, file names, publish steps, and voice. Without that, the activation energy to write something up is too high and the blog dies.
