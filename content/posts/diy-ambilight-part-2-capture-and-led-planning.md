---
title: "DIY Ambilight Part 2: Capture Working, LEDs Planned"
date: 2026-05-03
draft: true
tags: ["raspberry-pi", "homelab", "led", "hyperion", "hdmi"]
description: "HDMI splitter arrived. Got a clean 1080p30 frame out of the B101, Hyperion preview working, and LED wiring planned. 1080p60 is still broken."
---

The splitter arrived. This session was about proving out the full capture path — signal through the splitter to both the TV and the B101, a real frame captured and converted to PNG, and Hyperion's live preview running against actual Apple TV content. No physical LEDs yet, but the plan is locked.

*(Pictures coming — capturing screenshots mid-session was annoying enough without also photographing the setup.)*

## Signal chain verified

After wiring the Apple TV through the splitter, the first step was telling the B101 what kind of HDMI signal to expect. Setting the EDID caused a brief display dropout — expected, it's the HDMI re-handshake:

```bash
v4l2-ctl -d /dev/v4l-subdev0 --set-edid=type=hdmi,pad=0
```

After that the B101 reported a stable 1080p60 timing:

```bash
v4l2-ctl --query-dv-timings -d /dev/video0
```

```
Active width: 1920
Active height: 1080
Pixelclock: 148500000 Hz (60.00 frames per second)
```

Applied it:

```bash
v4l2-ctl --set-dv-bt-timings query -d /dev/video0
```

Everything looked right at the V4L2 level. Video input OK, format set to UYVY 4:2:2, media graph showing TC358743 → unicam path enabled. Then tried to actually capture something.

## 1080p60 failed at STREAMON

```bash
v4l2-ctl -d /dev/video0 \
  --set-fmt-video=width=1920,height=1080,pixelformat=UYVY \
  --stream-mmap=3 \
  --stream-count=1 \
  --stream-to=/tmp/frame.uyvy
```

```
VIDIOC_STREAMON returned -1 (Invalid argument)
```

The output file was created but zero bytes. `dmesg` explained why:

```
Device has requested 3 data lanes, which is >2 configured in DT
```

The TC358743 wants 3 CSI data lanes for 1080p60. The Pi 3's CSI interface only has 2 configured in the device tree. Signal detection worked fine — locking to 1080p60 timings is a chip-level thing. Actual streaming through the unicam pipeline is where it fell apart.

## Fix: switch Apple TV to 1080p30

Under **Settings → Video and Audio → Resolution → Other Resolutions**, switched the Apple TV to 1080p SDR 30Hz. The B101 immediately reported the lower clock:

```
Pixelclock: 74250000 Hz (30.00 frames per second)
```

Half the pixel clock, fits in 2 CSI lanes. Capture succeeded:

```bash
v4l2-ctl --query-dv-timings -d /dev/video0
v4l2-ctl --set-dv-bt-timings query -d /dev/video0

v4l2-ctl -d /dev/video0 \
  --set-fmt-video=width=1920,height=1080,pixelformat=UYVY \
  --stream-mmap=4 \
  --stream-count=1 \
  --stream-to=/tmp/frame.uyvy
```

```
-rw-rw-r-- 1 derek derek 4.0M May 3 00:21 /tmp/frame.uyvy
```

4.0 MB is exactly right: 1920 × 1080 × 2 bytes = 4,147,200 bytes. One full UYVY frame.

## Converting the raw frame to PNG

```bash
ffmpeg -y -f rawvideo \
  -pixel_format uyvy422 \
  -video_size 1920x1080 \
  -i /tmp/frame.uyvy \
  -frames:v 1 -update 1 \
  /tmp/frame.png
```

The PNG was produced but the image was mostly green — visually corrupted. The frame size was correct so the capture path was working, but the first frame after stream startup wasn't clean.

## Fix: capture 10 frames, extract the 9th

```bash
v4l2-ctl -d /dev/video0 \
  --stream-mmap=4 \
  --stream-count=10 \
  --stream-to=/tmp/capture10.uyvy
```

One thing worth noting: `v4l2-ctl` doesn't produce numbered files. The `%03d` pattern in `--stream-to` is treated as a literal filename — the output is one concatenated 40 MB file (10 × 4,147,200 bytes).

Extract frame 9 from the concatenated file:

```bash
ffmpeg -y -f rawvideo \
  -pixel_format uyvy422 \
  -video_size 1920x1080 \
  -i /tmp/capture10.uyvy \
  -vf "select=eq(n\,9)" \
  -frames:v 1 -update 1 \
  /tmp/frame-last.png
```

That produced a clean image. The TC358743/unicam pipeline needs a few frames after stream start before the output is reliable — the first frame is junk, a later one is fine.

Copy it off the Pi to check:

```bash
# run from Mac
scp derek@hyperion.local:/tmp/frame-last.png ~/Desktop/hyperion-frame-last.png
```

## Hyperion dashboard and preview

After confirming the raw capture was clean, started Hyperion and opened the dashboard in Chrome. Live video preview worked — the virtual LED border was reacting to the Apple TV content, which confirmed the full capture-to-color pipeline inside Hyperion.

There was an apparent artifact at the top of the preview that looked like a capture issue. It wasn't — it was the LED number overlay that Hyperion renders in the preview UI. The raw frame check confirmed the B101 capture itself was clean.

A few non-blocking log messages:

```
libcec.so.6: cannot open shared object file: No such file or directory
Failed loading libCEC library. CEC is not supported.
LED mapping area(s) have a huge number of pixels to be processed.
Every 2 pixels will be skipped to improve performance.
```

CEC isn't being used. The LED mapping warning is about processing load, not a capture issue.

## LED layout

Hyperion's default virtual layout had 86 LEDs on top and bottom, 48 on each side — 268 total. Hyperion estimated max current at 17.7A, which is about 88W at 5V. Definitely not powering that from the Pi.

Physically measured the TV and adjusted:

| Edge | Count |
|------|-------|
| Top | 80 |
| Bottom | 80 |
| Left | 48 |
| Right | 48 |
| **Total** | **256** |

With this layout, corner positions:

```
0   = top-left
80  = top-right
128 = bottom-right
208 = bottom-left
```

The data input will be at bottom center, which is position 168 (128 + 40).

## Wiring plan

Driving the LEDs directly from the Pi GPIO for now — no WLED controller yet. External 5V PSU for the strip.

```
Pi GPIO (data) → LED strip DIN
Pi GND         → LED strip GND
PSU +5V        → LED strip +5V (both ends)
PSU GND        → LED strip GND (both ends)
```

Pi GPIO is 3.3V, most SK6812 strips prefer 5V data. For a short data run it usually works, but if there's flicker the fix is a 74AHCT125 or 74AHCT245 level shifter between the Pi and the strip.

Power injection at both ends of the run. Starting at 10–20% brightness — full white at full brightness with 256 LEDs is not a test you run without a reason.

## Corner handling

Strips shouldn't be bent sharply at corners — it stresses the copper and pads. Plan is to cut at each corner's marked cut point and bridge with short 3-wire jumpers:

- Previous segment DOUT → next segment DIN
- +5V → +5V
- GND → GND

Add strain relief so the solder pads aren't mechanically loaded.

## Pilot section first

Before committing to the full install, starting with the bottom-left section only (~40 LEDs, ~2.5 ft). Tests to run before going further:

1. Solid red, green, blue — confirm color order
2. Brief white at low brightness — confirm power setup
3. Chase effect — confirm strip direction matches Hyperion layout

If the chase runs the wrong direction, use Hyperion's reverse setting rather than rewiring.

## Known-good capture config

| Setting | Value |
|---------|-------|
| Apple TV output | 1080p SDR 30Hz |
| B101 timing | 1920×1080p30 |
| Pixel clock | 74.25 MHz |
| V4L2 format | UYVY 4:2:2 |
| Frame size | 4,147,200 bytes |

1080p60 locks to timing but fails at STREAMON — CSI lane count issue with the Pi 3.

## Next

Set final Hyperion layout (80/80/48/48), input position 168, direction bottom-middle → bottom-left. Wire the pilot section. Test. If it works, cut corners and run the full strip.
