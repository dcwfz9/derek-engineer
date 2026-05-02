---
title: "Migrating derekwelty.com to Hugo + GitHub Actions"
date: 2026-05-02
draft: false
tags: ["hugo", "github-actions", "tooling", "web"]
description: "Replacing a Bootstrap 5 static site with Hugo and the Congo theme, deployed via GitHub Actions to GitHub Pages. Here are the gotchas."
---

[derekwelty.com](https://derekwelty.com) was a Bootstrap 5 single-page site built on the Start Bootstrap Freelancer template. It worked, but it hadn't been touched in years and updating it required wading through jQuery, a Gulp pipeline, and a bunch of hand-rolled CSS. Time to replace it.

The new stack: Hugo with the [Congo](https://jpanther.github.io/congo/) theme, deployed to GitHub Pages via GitHub Actions.

## Why Hugo

I already run this blog on Hugo + PaperMod via Netlify. The toolchain was familiar, the build times are fast, and Congo gave me a clean profile layout that actually looked like a personal site and not a blog template with the posts removed.

## The CI/CD setup

The repo (`dcwfz9/dcwfz9.github.io`) is a GitHub Pages repo. I wanted to keep the old site on `main` while building the new one on a `hugo-migration` branch, then flip over once it was ready.

That introduced a few non-obvious friction points.

### Workflow file placement

The workflow has to live on the branch it's triggering from. I initially put `.github/workflows/hugo.yml` on `main` with `branches: ["hugo-migration"]` and nothing fired. The fix: the file has to be on `hugo-migration` itself.

### Dart Sass via snap is broken in Actions

The standard advice for Hugo + Dart Sass on GitHub Actions is:

```yaml
- name: Install Dart Sass
  run: sudo snap install dart-sass
```

This fails silently or hangs. The snap daemon isn't reliable in Actions runners. Replaced it with a direct binary download:

```yaml
- name: Install Dart Sass
  run: |
    curl -L https://github.com/sass/dart-sass/releases/download/1.77.8/dart-sass-1.77.8-linux-x64.tar.gz | tar -xz
    echo "$(pwd)/dart-sass" >> $GITHUB_PATH
```

### Pin Hugo version to match local

The workflow pulls a specific `.deb` from Hugo's GitHub releases. Set `HUGO_VERSION` to whatever you're running locally — mismatches cause build failures that are annoying to debug in CI.

```yaml
env:
  HUGO_VERSION: 0.161.1
```

### Environment protection blocks the deploy

GitHub Pages has an "Environment" called `github-pages` with a branch allowlist. If your source branch isn't in that list, the deploy job fails with "not allowed to deploy to github-pages."

Fix: **Settings → Environments → github-pages → Deployment branches** — add `hugo-migration` (or whatever your branch is).

### Non-fast-forward push after rebasing

After fixing the above, I hit a non-fast-forward rejection when the remote had diverged. Standard rebase resolution:

```bash
git pull --rebase origin hugo-migration
git push
```

## The final workflow

```yaml
name: Deploy Hugo site to Pages

on:
  push:
    branches: ["hugo-migration"]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: false

defaults:
  run:
    shell: bash

jobs:
  build:
    runs-on: ubuntu-latest
    env:
      HUGO_VERSION: 0.161.1
    steps:
      - name: Install Hugo CLI
        run: |
          wget -O ${{ runner.temp }}/hugo.deb https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_withdeploy_${HUGO_VERSION}_linux-amd64.deb
          sudo dpkg -i ${{ runner.temp }}/hugo.deb

      - name: Install Dart Sass
        run: |
          curl -L https://github.com/sass/dart-sass/releases/download/1.77.8/dart-sass-1.77.8-linux-x64.tar.gz | tar -xz
          echo "$(pwd)/dart-sass" >> $GITHUB_PATH

      - name: Checkout
        uses: actions/checkout@v4
        with:
          submodules: recursive
          fetch-depth: 0

      - name: Setup Pages
        id: pages
        uses: actions/configure-pages@v5

      - name: Install Node.js dependencies
        run: |
          if [[ -f package-lock.json || -f npm-shrinkwrap.json ]]; then
            npm ci
          fi

      - name: Build with Hugo
        env:
          HUGO_CACHEDIR: ${{ runner.temp }}/hugo_cache
          HUGO_ENVIRONMENT: production
        run: hugo --minify

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v5
```

## One thing to remember

If you ever merge `hugo-migration` into `main`, update the workflow trigger:

```yaml
on:
  push:
    branches: ["main"]  # was hugo-migration
```

Otherwise pushes to main won't trigger a deploy and you'll spend five minutes wondering why the site isn't updating.
