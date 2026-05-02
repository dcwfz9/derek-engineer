---
title: "LaTeX Resume with Auto-Build on GitHub"
date: 2026-05-01
draft: true
tags: ["tooling", "latex", "github-actions"]
description: "How I write my resume in LaTeX and let GitHub Actions compile and publish the PDF on every push."
---

My resume lives in a git repo. Every time I push, a GitHub Action compiles it to PDF and drops it on a stable URL. No local toolchain needed after the initial setup, no manually uploading PDFs anywhere.

## Why LaTeX

Word docs drift. PDFs exported from Google Docs look fine until they don't. LaTeX gives me exact control over layout and is just a text file — diffable, versionable, no binary blobs.

Because it's plain text, it's also machine readable. Parsers, scripts, LLMs — anything that can read a file can read your resume. That's increasingly useful.

Git is version control by default. Every edit is tracked, every version is recoverable, and you can see exactly what changed between job applications.

I'm using the [Awesome-CV](https://github.com/posquit0/Awesome-CV) template, which handles all the styling. I scaffolded the initial file with [resumake.io](https://resumake.io), then checked the output into [dcwfz9/resume](https://github.com/dcwfz9/resume) and took it from there.

The compiler is `xelatex` — required by Awesome-CV for its font handling.

## The workflow

The GitHub Actions workflow does four things on every push:

1. Installs `texlive-xetex` and `texlive-fonts-recommended` on Ubuntu
2. Runs `xelatex resume.tex`
3. Uploads the PDF as a build artifact
4. Deploys it to an orphan `build` branch via [peaceiris/actions-gh-pages](https://github.com/peaceiris/actions-gh-pages)

```yaml
name: resume
on: [push]
jobs:
  paper:
    runs-on: ubuntu-latest
    env:
      DIR: .
      FILE: resume
    steps:
      - uses: actions/checkout@v3
      - name: Install TeXlive
        run: sudo apt-get update && sudo apt-get install texlive-xetex texlive-fonts-recommended -y
      - name: LaTeX compile
        working-directory: ${{ env.DIR }}
        run: xelatex ${{ env.FILE }}
      - name: move
        run: mkdir -p github_artifacts && mv ${{ env.DIR }}/${{ env.FILE }}.pdf ./github_artifacts/
      - uses: actions/upload-artifact@v4
        with:
          name: ${{ env.FILE }}.pdf
          path: ./github_artifacts

  deploy:
    needs: [paper]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/download-artifact@v4
        with:
          path: github_artifacts
      - name: move
        run: mkdir -p github_deploy && mv github_artifacts/*/* github_deploy
      - uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./github_deploy
          publish_branch: build
          force_orphan: true
```

The orphan `build` branch keeps compiled output completely separate from source. The latest PDF is always at:

```
https://github.com/dcwfz9/resume/blob/build/resume.pdf
```

That's a permanent link I can put anywhere — job applications, portfolio, whatever. It's always current.

## Building locally

```bash
sudo apt-get install texlive-xetex texlive-fonts-recommended
xelatex resume.tex
```

The CI loop is fast enough that I mostly just push and check the Actions tab.

## What I'd do differently

The workflow installs TeX packages fresh on every run, which takes ~2 minutes. Caching the apt layer or using a pre-built LaTeX Docker image would cut that significantly. Fine for now since I'm not updating the resume daily.
