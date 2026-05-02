---
title: "TRMNL X BYOS: Self-Hosting the Server Before the Device Arrives"
date: 2026-04-23
draft: false
tags: ["home-lab", "python", "networking"]
description: "Setting up a FastAPI BYOS server for the TRMNL X e-ink display, with local DNS and Caddy reverse proxy, before the hardware even ships."
---

I ordered a [TRMNL X](https://usetrmnl.com/) e-ink display to put on my desk. It's a 7.5" e-paper panel that polls a server for images and refreshes on a schedule. TRMNL has a cloud service, but I'm not paying a subscription for a display I can host myself — and BYOS (Bring Your Own Server) mode is the whole reason I bought the X model over the cheaper ones. The device hasn't arrived yet. The server is already running.

## The server

[This FastAPI BYOS server](https://github.com/rcarmo/python-fastapi-trmnl-server) handles everything: the firmware-facing API, plugin scheduling, image rendering, and a minimal web dashboard. The firmware protocol is simple:

```
TRMNL X firmware
  → GET /api/display
  → receives { image_url, filename, refresh_time }
  → fetches image_url → renders to e-ink panel
```

The `filename` alternates between `screen.bmp` and `screen1.bmp` each cycle so the ESP32 knows the image actually changed and doesn't skip the render.

### Plugins

Plugins live in `trmnl_server/plugins/`. Inherit `PluginBase`, set `AUTO_REGISTER = True`, output an image — the scheduler picks it up automatically. Each plugin generates both a 1-bit BMP (legacy firmware) and a grayscale PNG (newer firmware). The server handles dithering; the plugin just draws.

Plugins that ship with the repo:

| Plugin | What it renders |
|--------|----------------|
| `WeatherPlugin` | Minimalist weather card |
| `HNPlugin` | Top Hacker News headlines |
| `XKCDPlugin` | Latest xkcd |
| `ChartsPlugin` | Configurable charts |
| `RandomImagePlugin` | Image from a local pool |
| `CalibrationPlugin` | Grayscale calibration target |

### Running it

```bash
git clone https://github.com/rcarmo/python-fastapi-trmnl-server
cd python-fastapi-trmnl-server
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
make serve
```

Default port is `4567`. Useful commands:

```bash
# list registered plugins
python -m trmnl_server --list-plugins

# run one plugin in isolation (good for development)
python -m trmnl_server --run-plugin WeatherPlugin --plugin-output /tmp
```

State lives in `var/db/trmnl.db` — device registrations, playlist positions, battery samples, config overrides. Environment variables take precedence over anything set via the API, so `SERVER_PORT` in your shell always wins.

## Network setup

The device needs to reach the server by hostname on the local network. Not exposing this to the internet.

### DNS via AdGuard Home

[AdGuard Home](https://adguard.com/en/adguard-home/overview.html) is already running as my local DNS server. Adding a custom rewrite:

**Settings → DNS rewrites → Add rewrite**
- Domain: `trmnl.home`
- Answer: `192.168.4.47` (Mac mini's static LAN IP)

Any device using AdGuard for DNS can now reach `trmnl.home`. The TRMNL X will pick up the router's DNS, which points at AdGuard.

### Caddy reverse proxy

The firmware skips SSL by default to save battery. [Caddy](https://caddyserver.com/) proxies port 80 to the FastAPI server on 4567.

`Caddyfile`:
```
http://trmnl.home {
    reverse_proxy localhost:4567
}
```

```bash
caddy start --config ~/Code/trmnl/Caddyfile
```

`http://trmnl.home` → `localhost:4567`. No TLS, no auth — LAN only.

## What's next

Device arrives → BYOS setup:
1. Enable BYOS mode via device settings (requires Clarity Kit / Developer Edition firmware)
2. Point it at `http://trmnl.home`
3. Watch `/api/display` get hit in the server logs
4. Build a morning briefing plugin: today's calendar events, weather, any emails that need a reply

That last one is the point — same info I get from a Telegram briefing every morning, but always visible on the desk without picking up the phone.
