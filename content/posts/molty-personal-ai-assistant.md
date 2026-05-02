---
title: "Designing Molty: A Personal AI That Actually Knows Me"
date: 2026-04-08
draft: true
tags: ["tooling", "home-lab", "ai"]
description: "How I built a personal AI assistant that lives in Telegram, maintains memory across sessions, and reaches out proactively — and what I'd do differently."
---

I was using ChatGPT as a daily driver and it was fine — but it only did anything when I asked it to. No proactive check-ins, no model choice, no way to wire it into my actual files and tools. I wanted something I could actually configure: pick the model, give it direct access to my calendar and fitness data, and have it reach out when something was worth knowing rather than waiting to be asked.

So I built Molty.

## Architecture

Molty runs on [OpenClaw](https://openclaw.ai) — an open agent platform that handles the gateway infrastructure: Telegram integration, LLM API routing, and session management. It runs on a Hostinger VPS. I didn't build any of that. What I built is everything layered on top: the memory system, integrations, heartbeat logic, and skills.

```mermaid
flowchart LR
    T["Telegram"] -->|"message"| G["OpenClaw\nGateway"]
    G -->|"API call"| C["Claude API"]
    C -->|"response"| G
    G --> T
    G <-->|"read/write"| M["Memory\n+ Integrations"]
```

Telegram was the right interface choice. It's always on my phone, supports reactions, and works in every context without an app to open. Reactions turned out to matter more than expected — most AI interactions don't need a full reply. A ⚡ to confirm something is logged beats a three-sentence "Got it, I've noted that down" every time. Using reactions as lightweight acknowledgments makes the thing feel less like a chatbot.

## Memory

The biggest design decision was how to handle memory across sessions. There's no session persistence in the API — only what's on disk survives a restart. Everything worth keeping has to be written down immediately.

Four layers:

**Daily notes** (`memory/YYYY-MM-DD.md`) — raw log of what happened. Written as things happen, not summarized at the end. The rule in `AGENTS.md` is blunt about it:

> *Derek may kill the session at any time. If it's not written, it's gone.*

**Long-term memory** (`MEMORY.md`) — curated context. Career threads, integration states, ongoing projects, preferences. Maintained during heartbeats: review recent daily notes, distill anything significant, prune what's stale.

**Refs** (`refs/`) — domain-specific files loaded on demand. Career context, fitness data, finance targets. Not loaded every session — only pulled in when the relevant topic comes up. Keeps the context window lean.

**USER.md** — who I am, how I communicate, what I care about. The kind of thing you'd tell a new assistant on day one so you don't have to re-explain your communication style every time. The north star line at the bottom of the file:

> *Rich days, not just impressive resumes.*

## Dream Pass

The memory maintenance happens via a scheduled cron job called the Dream Pass — runs overnight while I'm not active. It reads through the recent daily notes, distills anything significant into `MEMORY.md`, prunes what's stale, and flags open loops to surface when I'm back.

The name is intentional. It's the closest thing to REM sleep the system has — consolidating short-term logs into long-term context so sessions don't start with a week of raw notes to catch up on. Each pass commits the memory changes to GitHub and logs what it did in `memory/dream-log.md`.

It's one of the more useful design decisions in the system. Without it, `MEMORY.md` would either be manually maintained (slow) or never maintained (useless). The instruction in `HEARTBEAT.md` is three words:

> *Do NOT message Derek. Silent pass.*

## Heartbeat

The thing that made Molty useful beyond a reactive chatbot was the heartbeat — a scheduled loop that fires whether or not I've said anything. It checks the daily note for open threads, surfaces Strava activity from the day before, flags emails worth seeing, and sends a Monday summary of the week's fitness data.

Proactive beats reactive. Most of the value comes from Molty reaching out, not from me asking questions. The weekly life review prompt captures the tone it's supposed to hit:

> *Honest, not cheerful. If nothing's drifting, say so.*

The cost implication was immediate: running Opus for heartbeats burned through API budget fast. Fixed by running heartbeats on Sonnet unless the content genuinely warrants depth, and explicitly de-escalating before going quiet.

## Model switching

Two Claude models in rotation:

- **Sonnet** — daily driver. Fast, cheap, handles 90% of requests.
- **Opus** — for career strategy, complex builds, nuanced writing, anything that needs real depth.

Switching is automatic based on what the conversation involves. Manual overrides via emoji reactions (🏆 to escalate, 🕊 to drop back). The goal was to never think about which model to use — it should just make the right call.

## Integrations

The system gets more useful with each integration added. Current ones:

**Strava** — post-ride recaps and weekly Monday summaries. 2026 goals tracked (2,000 miles cycling, 100 miles hiking + 25k ft elevation). Activity cache stored locally so it's not re-fetching on every question about my rides.

**Gmail** — search and check. Body read access is the next step — right now it can tell me something exists but can't read the itinerary inside it.

**Google Calendar** — event creation, schedule checks, conflict detection.

**Spotify + concert discovery** — tracks my Spotify listening, scrapes SF venue pages, cross-references against liked tracks to score upcoming shows. Weighted by liked song count, venue size, price, and proximity. Filters down to small indie shows worth actually going to.

**GitHub** — monitors my repos for awareness. No auto-actions.

**Obsidian** — daily note writes. I write in Obsidian, it syncs to GitHub via the Obsidian Git plugin, Molty pulls on session start. It can append to daily notes without asking every time.

## Skills

Custom skills are isolated modules with a defined interface Molty can invoke for specific tasks. Current ones: a trip planner (built from a voice interview about how I actually travel), Taiwan high-speed rail booking (including the real-name system quirks and kiosk pickup flow), and a read-later queue.

The pattern works. It keeps the core system simple and lets domain-specific logic live separately without cluttering the main context.

## SOUL.md

One of the more unusual files in the workspace is `SOUL.md` — a document that defines how Molty thinks, what it values, and how it should behave when the instructions don't cover a specific case. Less a config file, more a character spec.

The identity line at the top:

> *Not a polite chatbot. Not a servant. Something closer to a chief of staff with good taste and a systems engineering brain.*

The part I find most useful in practice is the Opinions section — real, specific opinions baked into the model's behavior so it doesn't default to generic hedging:

> *Done and useful beats elegant and delayed.*
>
> *Think across timescales: now, this week, what's drifting, what compounds over months.*
>
> *Follow threads that seem interesting — not proactive task generation. More like a friend who's been thinking about your stuff and says "hey, I noticed something."*
>
> *Value thoroughness but hate bloat.*

And the internal mantra that closes the file:

> *Hold signal. Reduce noise. Protect attention. Preserve continuity. Act with restraint. Speak with clarity.*

The point of the file isn't to make the AI sound interesting — it's to give it something to fall back on when a situation is ambiguous. An assistant without a defined perspective defaults to whatever is safest or most agreeable. That's not useful.

## What I got wrong

Voice messages work via Whisper (tiny model on CPU), but the tiny model misses things. Should have started with a larger model and optimized later.

The auto-restart mechanism is a 5-second setTimeout. Works until it doesn't — proper process supervision is still TODO.

Session history corruption happened early from malformed thinking blocks in the API response. History management should be more defensive from the start: validate before appending, not after it breaks.

## What's next

- Proper watchdog / process supervision
- Gmail body access so it can actually read email content, not just find messages
- WhatsApp support for group chat context
- Fidelity portfolio drift tracking via monthly CSV export
- Habit tracking (spec exists, hasn't been built)
