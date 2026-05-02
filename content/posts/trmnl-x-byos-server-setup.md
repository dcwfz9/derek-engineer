---
title: "TRMNL X BYOS: Self-Hosting the Server Before the Device Arrives"
date: 2026-04-23
draft: false
tags: ["home-lab", "python", "networking", "home-assistant"]
description: "Setting up a FastAPI BYOS server for the TRMNL X e-ink display, with local DNS and Caddy reverse proxy, before the hardware even ships."
---

This is a working log. The TRMNL X is ordered but hasn't arrived yet. The server is running.

## What is TRMNL X

TRMNL makes e-paper displays designed to sit on a desk and cycle through widgets — weather, calendar, news, whatever you build. The X model supports BYOS (Bring Your Own Server), which lets you point the device at your own backend instead of their cloud. That's the only mode I'm interested in.

The Clarity Kit ($25 add-on) includes the Developer Edition firmware and a battery upgrade. The Developer Edition is what exposes the BYOS endpoint in the device settings.

## The server

I'm running [this FastAPI BYOS server](https://github.com/rcarmo/python-fastapi-trmnl-server), a nearly-from-scratch rewrite of an earlier Flask implementation. It handles the firmware-facing API, plugin scheduling, image rendering, and a minimal web dashboard.

Architecture at a glance:

```
TRMNL X firmware
  → GET /api/display
  → receives { image_url, filename, refresh_time }
  → fetches image_url → renders to e-ink panel
```

The `filename` field changes on every refresh cycle so the ESP32 firmware knows the image is new and doesn't skip the render. The server alternates between `screen.bmp` and `screen1.bmp` to force cache-busting without any coordination with the device.

### Plugin system

Plugins live in `trmnl_server/plugins/`. Any class that inherits `PluginBase` with `AUTO_REGISTER = True` gets picked up automatically by the scheduler. Each plugin outputs two files — a 1-bit monochrome BMP (for legacy firmware) and a grayscale PNG (for newer firmware that supports it). The server handles dithering and grading; the plugin just generates an image.

Current plugins in the repo:

| Plugin | What it renders |
|--------|----------------|
| `WeatherPlugin` | Braun-inspired minimalist weather card |
| `HNPlugin` | Top Hacker News headlines |
| `XKCDPlugin` | Latest xkcd strip |
| `ChartsPlugin` | Configurable chart rendering |
| `RandomImagePlugin` | Image from a local pool |
| `CalibrationPlugin` | Grayscale calibration target |

The scheduler runs each plugin on its own interval and writes assets to `var/generated/`. The firmware just polls `/api/display` and gets back whatever is current in the rotation.

### State persistence

SQLite at `var/db/trmnl.db` tracks:
- Device registration and per-device playlist state
- Plugin rotation position
- Battery voltage samples
- Config entries (overrides env vars, survives restarts)
- Request logs

Config precedence: environment variables win over anything written via the `/settings/*` API, so `SERVER_PORT` in your shell always wins, but API-driven tweaks like display brightness or refresh intervals persist across restarts.

### Running it

```bash
cd ~/Code/trmnl
source .venv/bin/activate
python3 -m trmnl_server
# or: make serve
```

Default port is `4567`. List active plugins:

```bash
python -m trmnl_server --list-plugins
```

Debug a single plugin without starting the full server:

```bash
python -m trmnl_server --run-plugin WeatherPlugin --plugin-output /tmp
```

## Network setup

The device will be on the local network and needs to reach the server by hostname. I'm not exposing this to the internet — it's LAN-only.

### DNS via AdGuard Home

AdGuard Home is already running as the local DNS server (replaces Pi-hole). Adding a custom rewrite took 30 seconds:

**Settings → DNS rewrites → Add rewrite**
- Domain: `trmnl.home`
- Answer: `192.168.4.47` (the Mac mini's static LAN IP)

Now any device on the network that uses AdGuard as its DNS resolver can reach `trmnl.home`. The TRMNL X will use the router's DNS, which points at AdGuard, so it'll resolve correctly.

### Caddy reverse proxy

The firmware expects HTTP (it avoids SSL by default to save battery). Caddy handles the reverse proxy from port 80 to the FastAPI server on 4567.

`Caddyfile`:
```
http://trmnl.home {
    reverse_proxy localhost:4567
}
```

Start Caddy:
```bash
caddy start --config ~/Code/trmnl/Caddyfile
```

That's it. `http://trmnl.home` → `localhost:4567`. No TLS, no auth — this is a private LAN server.

## What's next

Device arrives → BYOS setup:
1. Put TRMNL X in BYOS mode via device settings (Clarity Kit / Developer Edition firmware enables this)
2. Point it at `http://trmnl.home`
3. Verify `/api/display` is getting hit in the server logs
4. Build a morning briefing plugin: calendar events + weather + any flagged emails

The morning briefing plugin is the whole point. The e-ink panel will show today's schedule and conditions at a glance without touching a phone.
