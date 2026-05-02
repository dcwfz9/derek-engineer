---
title: "How This Blog Was Built (and Who's Writing It)"
date: 2026-05-01
draft: true
tags: ["hugo", "netlify", "claude-code", "meta"]
description: "Setting up a Hugo blog with PaperMod, deploying via Netlify, and wiring up Claude Code as the publishing agent."
---

This is a breakdown of how derek.engineer works — the stack, the setup, and why publishing a post sometimes means just describing what I built to an AI.

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

I wanted a blog, not a project. That meant:

- No frontend build pipeline to maintain
- Markdown as the writing format
- Git-backed history
- Fast local preview

Hugo fits. `hugo server --buildDrafts` spins up a live-reloading local preview in under a second. `hugo` builds the full site in milliseconds. There's nothing to babysit.

PaperMod was the obvious theme choice — minimal, dark mode by default, readable code blocks, and well-maintained. I didn't need a custom design, I needed a sane starting point.

## Netlify setup

The `netlify.toml` pins the Hugo version so the production build matches local:

```toml
[build]
  command = "hugo"
  publish = "public"

[build.environment]
  HUGO_VERSION = "0.161.1"
```

Netlify picks this up automatically. New commit to `main` → deploys. No CI to configure, no Actions YAML, no deploy scripts. It just works.

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

## The interesting part: Claude Code as the publishing agent

The part that makes this setup different is `CLAUDE.md`.

Every Hugo project can have a `CLAUDE.md` file that instructs Claude Code how to work in that repo. Mine tells it exactly how to create and publish posts:

- Front matter format with required fields
- File naming convention (kebab-case, descriptive)
- The three-command publish workflow (`git add`, `git commit`, `git push`)
- Voice and tone guidelines

The result: when I finish a project, I can describe what I built in a conversation and Dex (my Claude Code instance) writes, formats, and publishes the post. I review it, maybe tweak a line or two, and it's live.

That's how most of the posts on this site were written. I ran the build, described what happened, and Dex turned it into a post.

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

## What I'd do differently

Nothing significant. This setup is boring in the right way — it doesn't compete for attention. The hard part of a blog is writing, not the infrastructure. Hugo + Netlify + Markdown in GitHub makes that as frictionless as I've found.

The `CLAUDE.md` integration is the one thing I'd add to any similar project immediately. Lowering the activation energy for publishing is the most important factor in whether a blog actually gets posts.
