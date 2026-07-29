---
title: "Using an OTA TV Antenna as a Rain Gauge (Once It Actually Rains)"
date: 2026-07-29
draft: true
tags: ["home-lab", "hardware", "rf", "data"]
description: "I set up an RF signal logger to detect rain from UHF attenuation. San Francisco had other ideas."
---

The concept is real. UHF radio signals attenuate in rain. A consistent broadcast at a fixed frequency will drop measurably when precipitation is in the path between the transmitter and the antenna. Compare it against a VHF channel — which is much less affected by rain — and you can separate weather effects from transmitter issues. Same principle as dual-band GPS receivers canceling ionospheric delay by differencing L1 and L2.

I have an HDHomeRun FLEX DUO on the network with two tuners. I'm already using tuner 0 for TV. I dedicated tuner 1 to logging signal strength every 5 minutes across three channels: ch28 UHF 557 MHz (strong reference), ch31 UHF 575 MHz (moderate, more dynamic range), and ch12 VHF 207 MHz (control, rain-insensitive). I paired that with NWS observations from KSFO pulled every sweep. The code is about 100 lines of Python.

This should work.

Here is the problem.

It has not rained in San Francisco since April.

---

I have 1,119 rows of data spanning July 17 to July 22. Precipitation column: all zeros. The signal strength on the UHF channels sits between 91 and 100 out of 100, nearly pegged to the ceiling. There is nothing to attenuate because there is no rain. There is nothing to see in the data because the data is five days of "everything is fine, it is 65°F and sunny, please stop watching this."

This is not a hardware problem or a software problem. It is a calendar problem. I built a rain sensor in a city that does not have rain from May through October.

San Francisco's wet season starts in November. I have logged this, set a reminder, and will check back then. If the antenna picks up the first real storm of the season before the weather app does, I will be unreasonably happy about it.

---

*More when it rains. Signal logger: [`hdhr_weather.py`](https://github.com/dcwfz9/dex). NWS station: KSFO. Channels: UHF 557 MHz, UHF 575 MHz, VHF 207 MHz.*
