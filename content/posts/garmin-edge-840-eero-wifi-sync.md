---
title: "Getting My Garmin Edge 840 to Sync Over eero Wi-Fi Again"
date: 2026-10-16
draft: true
tags: ["raspberry-pi", "networking", "garmin", "home-lab", "hardware-in-the-loop"]
specs:
  device: "Garmin Edge 840 Solar, a Raspberry Pi 5 wired to my eero network"
  os: "Raspberry Pi OS (Debian 13), NetworkManager 1.52"
  tools: "nmcli, wpa_supplicant in access point mode, dnsmasq, nftables, Claude Code"
description: "For a year my Edge 840 only synced over Wi-Fi right after I re-added the network. A 2.4 GHz-only network from a Raspberry Pi 5 got it syncing again, and the Pi's logs show what the Edge actually does."
---

For about a year my [Edge 840 Solar](https://www.garmin.com/en-US/p/798946/) hasn't synced over Wi-Fi at home. If I delete the network on the Edge and add it again, it syncs once. After that it sits on "connecting" and never transfers anything, even when I press Wi-Fi Sync.

I want to bring the bike home, stop the ride, and find it on Strava and Garmin Connect without touching my phone. That's how it used to work. I've worked at both Garmin and eero and still have friends at both, so I also wanted real data on whose bug this is instead of a guess.

<!-- PHOTO: the Pi 5 next to the eero, or the Edge showing a finished Wi-Fi sync. -->

## What the network looks like

My Wi-Fi is eero: two access points, my main network, and a guest network I'd named "Garmin Sync" in an earlier attempt to fix this. Claude scanned from the Pi: both names are on 2.4 GHz channel 1 and 5 GHz channel 116, from both access points, WPA2 only (no WPA3). So WPA3 transition mode, the usual suspect for Garmin Wi-Fi trouble, wasn't it.

The Edge's radios are 2.4 GHz only: its [spec page](https://www8.garmin.com/manuals/webhelp/GUID-16B12CFE-F96E-4DE9-9F5F-8C4A5936D3B9/EN-GB/GUID-62D44DE6-9C4D-4834-8267-8B23F891A9F3.html) lists one wireless frequency, 2.4 GHz at 19.5 dBm. eero puts one network name across every band by design, and the only 2.4 GHz-only option is [a 10-minute pause](https://support.eero.com/hc/en-us/articles/360049983772) meant for setting up new devices ([no permanent split](https://support.eero.com/hc/articles/115005497223)). My guest network had the same shape as the main one, which is probably why it didn't help.

Other Edge 840 owners describe the same thing (none of them say what router they have):

1. [It works right after the network is re-added, then not after the Edge is switched off](https://forums.garmin.com/sports-fitness/cycling/f/edge-840-series/415456/my-edge-840-won-t-reconnect-to-wifi-automatically). The fix one owner reported: a 2.4 GHz-only network name.
2. [Auto upload quit after about a year](https://forums.garmin.com/sports-fitness/cycling/f/edge-840-series/418459/840-solar-will-not-sync-via-wifi), and a 2.4 GHz-only guest network brought it back.
3. [On 30.18 (2026)](https://forums.garmin.com/sports-fitness/cycling/f/edge-840-series/432386/wifi-sync-always-fails-on-30-18), re-adding the network "seems only to work for a while." No reply from Garmin in any of them.

## A 2.4 GHz network on the Pi

eero can't do a 2.4 GHz-only network, so the Edge needed one from somewhere else. I went through the hardware I had: an ESP32-S3-ETH (busy as my 433 MHz gateway), a plain ESP32 (saved for the TV backlight), the Mac mini, the UNO Q (no Ethernet) and the Pi 5. The Pi 5 won: it's wired to the eero, its Wi-Fi radio wasn't doing anything, and Linux gives me logs. An ESP32 NAT router would have worked too, but with one radio its network has to follow the eero's channel.

```mermaid
flowchart TD
    EDGE["Edge 840 Solar"] -->|"2.4 GHz only, ch 6, WPA2"| PI["Raspberry Pi 5\nWi-Fi access point"]
    PI -->|"NAT out Ethernet"| EERO["eero home network"]
    EDGE -.->|"before: same name on\n2.4 and 5 GHz, syncs once"| EERO
    EERO --> GC["Garmin Connect"]
    GC --> ST["Strava"]
```

Claude wrote a setup script for the Pi. What it sets up:

| | |
|---|---|
| Network name | `Garmin Sync 2.4`, 2.4 GHz only |
| Channel | 6 (the eero is on 1, the neighbors on 9 and 11) |
| Security | WPA2-PSK, AES only, no management frame protection, no fast roaming |
| Addresses | IPv4 only, NAT out the Pi's Ethernet, clients get 10.42.0.x |
| Firewall | clients get the internet and nothing else: no home network, nothing on the Pi but DHCP and DNS |
| Logs | every join, DHCP lease and DNS lookup, plus byte counters for what the Edge sends and receives |

The core of it is one [NetworkManager](https://networkmanager.dev/docs/api/latest/nmcli.html) connection:

```bash
nmcli con add type wifi ifname wlan0 con-name garmin-sync-ap ssid "Garmin Sync 2.4" \
  802-11-wireless.mode ap 802-11-wireless.band bg 802-11-wireless.channel 6 \
  802-11-wireless.powersave disable \
  802-11-wireless-security.key-mgmt wpa-psk 802-11-wireless-security.psk "<passphrase>" \
  802-11-wireless-security.proto rsn 802-11-wireless-security.pairwise ccmp \
  802-11-wireless-security.group ccmp 802-11-wireless-security.pmf disable \
  ipv4.method shared ipv6.method disabled
```

and an nftables table that loads before NetworkManager at boot, so the network never comes up without it:

```
table inet garmin_sync_ap {
	chain input {
		type filter hook input priority filter; policy accept;
		iifname "wlan0" ct state established,related accept
		iifname "wlan0" udp dport { 53, 67 } accept
		iifname "wlan0" tcp dport 53 accept
		iifname "wlan0" icmp type echo-request accept
		iifname "wlan0" counter drop
	}
	chain forward {
		type filter hook forward priority filter; policy accept;
		iifname "wlan0" ip daddr { 10.0.0.0/8, 100.64.0.0/10, 169.254.0.0/16, 172.16.0.0/12, 192.168.0.0/16 } counter drop
		iifname "wlan0" counter comment "uploaded by AP clients"
		oifname "wlan0" counter comment "downloaded by AP clients"
	}
}
```

Raspberry Pi OS keeps Wi-Fi blocked until a country is set, so the script also sets it to US.

Before the Edge touched it, Claude checked the network from my Mac mini's Wi-Fi: an address from the Pi, Garmin Connect loading, 48 Mbit/s down and 20 up (curl against Cloudflare's speed test), and Home Assistant timing out through the Pi while it loaded fine over Ethernet. Then a reboot, to make sure the firewall comes up before the network does.

## A "connection failed" that wasn't

The first time I added the network, the Edge said "connection failed." I figured it was a flaky connection and tried again: same message. Then I switched to a simpler password to rule out a typo, and still got "connection failed." The Pi's log told a different story: a few minutes later the Edge connected, got an address and looked up Garmin's servers (the 12:02 am row below), and a Wi-Fi Sync after that worked. It happened again the next morning: "connection failed," then the Pi's log showed it connected a minute later.

Just an artifact I found along the way, but worth knowing: the message on the Edge isn't proof the connection failed. Garmin has fixed a cousin of this before, in 27.13: "Fixed manual Wi‑Fi sync incorrectly reporting sync failure."

## What the Edge does on a network it likes

From the Pi's logs (times are mine, Pacific; the byte counters came in after the first session):

| When | What started it | Connected for | Up | Down | Garmin hosts it looked up |
|---|---|---|---|---|---|
| 12:02 am | first good join | 16 s | not counted | not counted | `clock`, `api.gcs`, `omt`, `download` |
| 5:31 am | re-added the network | 3 min 13 s | about 1.6 MB | about 55 MB | `clock`, `gold`, `api.gcs`, `download` |
| 5:41 am | nothing I pressed: the Edge restarting after a software update | 105 s | 243 KB | 942 KB | `connectapi`, `services`, `api.gcs`, Connect IQ, and `www.strava.com` |
| 7:41 am | Wi-Fi Sync, after a short trainer ride | 30 s | 181 KB | 804 KB | `clock`, `gold`, `download` |

The 55 MB was a software update. A ride I hadn't synced yet went up in the 5:31 session and the trainer ride in the 7:41 one, both showing up in Garmin Connect.

Two things changed. The Edge reconnected on its own after a restart, which is the exact step that fails on eero. And Wi-Fi Sync finishes now; on eero it got stuck connecting.

One thing didn't. After I saved the trainer ride, the Edge didn't try Wi-Fi at all: nothing in the Pi's log until I pressed Wi-Fi Sync. The manual only says auto upload sends the ride "as soon as you finish recording." Owners on the forums say the Edge uploads through the phone over Bluetooth instead when the phone is connected. I haven't tested whether that's what happened.

A detail for the Wi-Fi people: NetworkManager's access point offers both WPA2-PSK and PSK-SHA256 (it adds SHA256 even with management frame protection off), and the Edge picks SHA256: its handshake messages carry key descriptor version 3. I haven't checked what eero offers, or whether it matters.

## The 5 GHz test

Before blaming the shared name, I wanted to know whether the Edge can see 5 GHz at all. Claude moved the Pi's network to 5 GHz only (channel 36), same name and password, and checked it was up by joining it from my Mac on channel 36. Then I pressed Wi-Fi Sync on the Edge.

The Edge sat on "connecting" for a while, then said "sync failed." The Pi's log has nothing from the Edge in that window: not one association attempt. So it can't use a 5 GHz network even with the right name and password, which matches its spec page.

How it failed is the interesting part. On eero, Wi-Fi Sync sat on "connecting" and never changed (I never waited long enough to see it time out). Here it gave up with "sync failed." My guess is that on eero the Edge does find the network and gets stuck partway through connecting, instead of not finding it at all. That's from the message alone, not a capture.

## Garmin bug or eero bug

Mostly Garmin, as far as I can tell. My eero networks are ordinary WPA2 on channel 1, which every other device in the apartment handles, and the Edge joins them fine right after the network is added. It's reconnecting later that fails. Garmin's own [Edge 840 change logs](https://forums.garmin.com/sports-fitness/cycling/f/edge-840-series/422288/edge-840-series-version-28-34) keep fixing Wi-Fi: 25.24 "Fixed adding a password protected WiFi network" and 27.13 "Fixed manual Wi‑Fi sync incorrectly reporting sync failure." The owners above have the same pattern, and none of them mention eero.

eero's part is a design choice, not a bug: one name for every band, and no permanent 2.4 GHz-only network. That removes the workaround every other Edge owner uses.

eeroOS 7.17.2 (October 7) says "Fixed an issue where certain devices couldn't connect with WPA2 encryption." It hasn't reached my network yet. My guess is it isn't this, since my problem is a year old and the Edge connects fine right after it's added, but that's untested.

If I could ask each company for one thing: Garmin, reconnect to a saved network after a power cycle when the same name is also on 5 GHz or on more than one access point. eero, an optional 2.4 GHz-only network name for devices like this.

What I can't pin down from here is the exact trigger. The 5 GHz test makes the shared name across bands a strange suspect: the Edge can't see the 5 GHz half. My guess is the two access points sharing one name on channel 1, or eero steering the Edge between them, but that's a guess. The 2.4 GHz-only networks that fixed it for other owners probably changed several of those things at once, so their fix doesn't separate them either.

## Where it stands

Wi-Fi Sync works, and the Edge rejoins by itself after a restart. The real test hasn't happened yet: an actual ride, the Edge asleep in between, saved at home, on Strava without me touching anything.

Future work:

- A real ride, start to finish, with nothing pressed.
- Phone Bluetooth off versus on after saving, to see whether the phone path is why auto upload didn't fire.
- Move the network off the Pi 5, which I have other plans for, to a Pi 3B I have lying around. Same name and password, so the Edge won't notice.
- When eeroOS 7.17.2 lands, put the Edge back on eero with the Pi's network off and repeat the test.
- Finish the reproduction: the Pi on 2.4 GHz plus a same-named 5 GHz network from the Mac, then two 2.4 GHz access points with one name, to see which one makes the Edge hang on connecting.
- A Wi-Fi capture on channel 1 while the Edge hangs on eero.

---

*[How this was built](/how-i-work/): Claude Code went through the forum reports and eero's docs, scanned my Wi-Fi, wrote the setup script and the firewall, and read the Pi's logs to see what the Edge was doing. I spotted the pattern (it only syncs right after re-adding the network), set the rule of using hardware I already had, picked the Pi 5, ran every test on the Edge (adding the network, Wi-Fi Sync, the trainer ride, the 5 GHz test), and pushed on whether this is a Garmin bug. Tested: the Edge joining the Pi's network, a software update, an unsynced ride and a trainer ride going up through it, the Edge reconnecting by itself after a restart, the Edge never trying to join a 5 GHz-only version of the network, the Pi coming back after a reboot, and, with my Mac as the client, speed and the firewall keeping clients off my home network. Not tested: auto upload right after saving a ride, a ride with nothing pressed, eeroOS 7.17.2, and what exactly breaks on eero.*
