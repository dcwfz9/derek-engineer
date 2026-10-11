---
title: "The Vornado EOS 9 on Home Assistant"
date: 2026-10-10
draft: false
tags: ["home-assistant", "esp32", "cc1101", "openmqttgateway", "mqtt", "rf", "flipper-zero", "hardware-in-the-loop"]
specs:
  device: "Waveshare ESP32-S3-ETH with a 433 MHz CC1101 module; Vornado EOS 9 fan"
  os: "OpenMQTTGateway built from source with local changes; Home Assistant 2026.7"
  tools: "PlatformIO, Mosquitto, ufbt and a Flipper Zero (official firmware 1.4.3), Claude Code"
description: "A Flipper Zero remote built from the decode, then an ESP32-S3 and a CC1101 running OpenMQTTGateway that put the Vornado EOS 9 on Home Assistant: a transmit path with no data line, a logging bug that dropped MQTT every few minutes, and one receiver for the fan's remote and the neighbors' weather sensors."
---

The [first post](/posts/vornado-eos-9-rf-remote-reverse-engineering/) ended
with a plan: a CC1101 and an ESP32 that put the fan on Home Assistant. Before
building it I wanted to know the decode was enough on its own, so the Flipper
Zero came first, as a stepping stone. Then the box.

What I wanted from the box: every button of the remote in Home Assistant,
automations that use them, and one cable under the desk for data and power.
The board has an Ethernet jack and takes a PoE module, which is what that last
part needs.

This is another piece of my mission to automate my life, and I over-engineer
on the way. There have been a lot of posts lately because I've been Claude
maxxing.

```mermaid
flowchart LR
    HA["Home Assistant\n+ Mosquitto"] -- MQTT --> G["ESP32-S3\ngateway"]
    G -- SPI --> R["CC1101\n433.92 MHz"]
    R -- decoded presses --> G
    R -- OOK --> F["Vornado EOS 9"]
    REM["the fan's remote"] -- OOK --> R
```

## The Flipper first

The Flipper Zero already has a sub-GHz radio, so I made it the fan's remote
before building anything. The question was whether every button press can be
made from the 32-bit word, with no captured file anywhere. I asked for it on
the bike ride home, with speed on Up and Down and oscillation on Left and
Right. Claude Code put the other four functions on long presses and wrote the
spec, and at my call a cheaper model wrote the app from it.

The app puts the 12-bit command under the 20-bit address and writes out the
timings, with T = 400 us: a 1T mark and a 30T gap for sync, then 3T on and 1T
off for a 1, 1T on and 3T off for a 0. A press is eight frames, about half a
second on air. Before it went on the Flipper, its frames had to match the file
that turned the fan on in the first post, value for value. They did.

![The Flipper screen with the app's legend: one row per key with its tap and hold. OK is Power and Timer, Up is Speed + and Speeds, Down is Speed - and Airflow, Left is Osc on/off and Lamp, Right is Osc 90/180 and Light. The status line reads Ready.](figs/flipper-legend.png)

On the fan, Power, the 12 hour timer and Selection + and - did what the map
says, and Lamp Mode toggles. The oscillation codes didn't. The first post named
0x040 and 0x080 as Vornado's 90 and 180 degree buttons. Sending 0x040 lights an
indicator with up and down arrows on the front panel, and sending it again
turns it off. Sending 0x080 goes to 90, and each press after that flips between
90 and 180. So it's one control that turns an oscillation on and off and one
that picks the range. I haven't watched which way the head moves for the first
one; the guide mentions a vertical oscillation control, and that's my guess.

The Flipper also showed the catch for the box: the fan never answers, and most
of its codes are toggles, so whatever controls it has to keep a guess of its
state.

## The box

The board is a Waveshare ESP32-S3-ETH (the PoE version: ESP32-S3R8, 16 MB
flash, 8 MB PSRAM, a W5500 for Ethernet). The radio is a generic 8-pin 433 MHz
CC1101 module. The firmware is OpenMQTTGateway, built from source with
PlatformIO, and Home Assistant gets the Mosquitto add-on, which I installed
for this.

The pins came from one constraint: the PoE module had to stay on, so the radio
could only use header pins it doesn't cover, from IO2 to the antenna corner.
GPIO 4 to 14 aren't on the headers at all (the TF slot and the W5500 use them).
That left exactly six pins, five signals and a ground, plus 3V3 from under the
module.
Claude Code matched the wires to the resistor colour code, so a wire's colour
is its CC1101 pin number:

| CC1101 pin | Signal | Wire | ESP32-S3-ETH |
|---|---|---|---|
| 1 | GND | brown | GND |
| 2 | VCC | red | 3V3 |
| 3 | GDO0 (data) | orange | IO44 |
| 4 | CSN | yellow | IO0 |
| 5 | SCK | green | IO1 |
| 6 | MOSI | blue | IO2 |
| 7 | MISO | violet | IO43 |
| 8 | GDO2 | none | |

The first wiring was press-fit jumpers, a known risk I took to get it online
before I left the house so the rest could be done remotely. It's soldered now,
and it will get rewired again into a printed case.

## Sending without the data line

OpenMQTTGateway's RF module sends by bit-banging the radio's data pin (GDO0)
with RCSwitch timing. The press-fit data wire wasn't making contact at first,
so Claude Code added another way, and it stayed: the CC1101 clocks the frame
out of its own FIFO. At 2.51 kBaud one symbol is one 398 us tick, so a frame is
just bits: the sync is a 1 and thirty 0s, a data 1 is `1110`, a 0 is `1000`,
and the stop mark is `10`, which the real remote sends and the Flipper app
leaves out. That's 161 symbols, three frames to a 63-byte FIFO
load, eight frames per press. Home Assistant sends it the same way it would any
OpenMQTTGateway code, with a protocol number I picked for it:

```
home/OMG_VORNADO/commands/MQTTto433   {"value": 56545282, "protocol": 99}
```

The timing is crystal-timed instead of bit-banged. The Flipper made a good
witness: stock `subghz rx_raw` caught the gateway's frames, and the same
decoder that checked the Flipper app sorted the pulses (no-action codes only,
so nothing at home moved).

| Flipper-measured | Gateway | Real remote |
|---|---|---|
| short mark | 406 us | 397 us |
| long mark | 1204 us | 1205 us |
| short space | 388 us | 388 us |
| long space | 1188 us | 1200 us |
| sync gap | 11.94 ms | 11.96 ms |
| spread (sd) | 14 to 19 us | about 21 us |

It decoded 7 of the 8 frames in every send; the Flipper's receiver misses the
first one. Then at home, Power turned the fan on, and again off.

## Dropping off every few minutes

The first run of all ten commands did nothing. Not one was acknowledged.

Mosquitto's log on Home Assistant had the pattern. The gateway would connect,
then 90 seconds later the broker dropped it with "exceeded timeout" (1.5 times
the 60 s keepalive), and a few minutes later it connected again. Sixteen of
seventeen sessions between 22:12 and 23:48 ended that way. Wi-Fi was the
obvious suspect, at -73 dBm.

It wasn't Wi-Fi. For two hours, from 00:04 to 01:59, a script held the board's
USB port open to catch a crash, and nothing dropped. Four minutes after the
script let go, the timeouts came back, every session until 04:41.

The board logs over the ESP32-S3's built-in USB serial. When the Mac is
attached but nothing reads the port, each write waits for room, up to 20 tries
of 100 ms. OpenMQTTGateway logs on every message, so the main loop stopped
getting back to MQTT in time. One line after `Serial.begin()` fixed it:

```cpp
Serial.setTxTimeoutMs(0); // unread log output is dropped instead of waited on
```

With the port attached and idle, the old build was dropped 2.5 minutes after
the port closed. The fixed one stayed up, and the broker's log has no drop like
that since; the timeouts left are from test builds and from unplugging it to
move it. On a wall charger it couldn't happen anyway, with no USB host to wait
on.

## Ethernet, with Wi-Fi behind it

OpenMQTTGateway has an Ethernet path, and Arduino-ESP32 3's `ETH.begin()` can
drive a W5500 from build flags alone: CS 14, INT 10, RST 9, SCK 13, MISO 12,
MOSI 11, on SPI3 so it never shares the radio's bus. The first boot came up at
100 Mbps. Wi-Fi stays configured as the fallback, and that got a real test:
when I moved the gateway to a room with no cable, it came back on Wi-Fi by
itself. Firmware updates go over the network now; a build with a new version
string booted from an OTA push in 25 seconds.

## Testing with the Flipper

The Flipper did most of the testing, and the reason is that it had already
proven it could talk to the fan: its remote app makes frames the fan obeys.
That made it a known-good reference in both directions. To test the
gateway's transmitter, a script told the gateway to send over MQTT and had the
Flipper's stock `subghz rx_raw` capture what went out on the air, then decoded
the pulses and compared them with the real remote's timing (the table above).
To test the receiver, the same script had the Flipper transmit a frame and
checked what the gateway published. Both ran from the Mac with the Flipper on
USB, so the whole loop, Home Assistant to MQTT to radio to air and back, could
be tested without me at the fan or at the keyboard. Anything that could move
the fan used the no-action codes, 0x400 and 0x800.

## Home Assistant

Each button is an MQTT-discovery entity under one device, named as in
Vornado's Quick-Start Guide. Two of those names are inferences from the first
post: Fan Speeds and Airflow Modes are the buttons I captured as `Blade` and
`Menu`. The two oscillation buttons are named for what the fan did when the
Flipper sent them. The buttons grey out when the gateway's last-will says it's
offline.

![The Fan dashboard in Home Assistant at phone width: ten buttons in two columns (Power, 12 Hour Timer, Selection minus and plus, Fan Speeds, Airflow Modes, Vertical Oscillation, Panoramic Oscillation 90/180, Lamp Mode, Light Settings), then an Automation card with the "Fan is on (assumed)" toggle, the apartment temperature at 65.2 F, a slider set to 77 F, time pickers set to 10:00 PM and 7:00 AM, and toggles for the six fan automations.](figs/fan-dashboard.png)

The fan reports nothing, and Power is a toggle. "Turn the fan off" can only
mean "press Power if it's on," so Home Assistant keeps a guess: a "Fan is on
(assumed)" helper that flips every time a Power command goes to the gateway,
whether it came from the dashboard, an automation or a script, and when the
gateway hears the real remote's Power (below). Four automations read it:

- **On at bedtime:** 10 PM, if I'm home and the guess says off, and again if I
  get home later in that night window. I've been trying to go to bed earlier,
  so the fan starting at 10 is the cue to head down.
- **Off in the morning:** 7 AM, but only if the guess says on.
- **Off when I leave:** ten minutes after I leave home (the same delay as my
  other leave-home automation, so a GPS blip doesn't count), it presses Power,
  but only if the guess says the fan is on.
- **On when the apartment is hot:** the living room sensor above 77 °F for five
  minutes, only while I'm home and the guess says off. The threshold is a
  slider; I said hot is somewhere between 75 and 79 °F.

Both schedule times are pickers on the dashboard. I used to leave the fan
running all day. What I want is the fan on while I'm asleep, which a motion
sensor in the bedroom would know and the clock only approximates, so the
sensor is on the list.

Three departures in one day, from Home Assistant's logbook:

| I left | The guess was | What happened |
|---|---|---|
| early afternoon | on | exactly ten minutes later, Power pressed, guess to off |
| late afternoon | off | nothing, which is what the guess is for |
| evening | on (set by hand) | exactly ten minutes later, Power pressed, the gateway acknowledged the transmit a second after that, guess to off |

I was out for all three. For the last one the fan was on when I left and off
when I got home. The hot rule hasn't fired; the apartment has stayed between
65 and 68 °F.

The 10 PM rule didn't fire that night, because my phone had died while I was
out, so Home Assistant never learned I was home. When I powered it back on,
Home Assistant saw me home, the arrival rule pressed Power in the same second,
and the gateway acknowledged the transmit a second later. A dead phone is
invisible to GPS and to its own Wi-Fi, so "home" is now broader. eero has no
official Home Assistant integration; a community one installs through HACS and
signs in with the account's email and a one-time code. It sees the phone the
moment it joins the Wi-Fi, and I added that tracker to my Home Assistant
person, so every presence rule, my older lamp automations included, counts me
home as soon as the phone is on the network. My Apple Watch is a second
tracker, which the schedule and hot rules use as a fallback, since a watch
joins the Wi-Fi by itself when its phone is away or dead. The cost is on the
way out: eero keeps a device home for five minutes after it disconnects, so the
leave rule can fire a few minutes later than it did on GPS alone.

## One receiver for the remote and the sensors

The guess goes stale the moment someone uses the real remote, so the gateway
has to hear it. By then the radio's receiver was running rtl_433, for the
neighbors' weather sensors (the next post). OpenMQTTGateway runs one decoder
library at a time, so Claude Code said it was the remote or the sensors.

I didn't buy that, and I was right. Both are 433.92 MHz on-off keying, so the
same radio hears both. The limit was software: rtl_433 has no decoder for the
fan. Claude Code wrote one in rtl_433's own format and added it to the
library's decoder list at startup with its `register_protocol()`, so the
library itself is untouched:

- pulse width modulation, 400 us and 1200 us marks
- a row breaks at the 12 ms sync gap, and two identical frames are required
- the command has to be one-hot, so other remotes can't trip it

A whole press arrives as one signal, so it comes out as one message,
`Vornado-Remote` with the button's name. A Home Assistant automation flips the
guess when the button is Power, ignoring repeats for two seconds.

With the Flipper alternating the fan's no-action code and a generic test
remote that a stock rtl_433 decoder reads, all four sends decoded on the same
receiver. Simulated presses checked the Home Assistant side: Power flips the
guess, a repeat inside two seconds doesn't, and other buttons leave it alone.

Then the real remote. Next to the gateway at 20:51 I pressed Power four times.
Each press came through as one message at -55 to -72 dBm and flipped the guess
within the same second: off, on, off, on, all four logged under "follow the
physical remote". From my bedroom, a couple of minutes earlier, the presses
didn't register. The gateway is at the kitchen's west window, a spot I picked for the
weather sensor, and it doesn't hear the remote from there. I'm fine with that.
The point of all this is to stop using the remote, I hadn't been using it
anyway, and the gateway only has to reach the fan to transmit, which it does.
The gap is that a press from out of its range goes unnoticed, the guess goes
stale, and the rules trust the guess. That's the one I'm living with.

## Not done yet

- Rewire it into a printed case, then put the PoE module on so it runs from
  one cable.
- A motion sensor in the bedroom, so the fan follows whether I'm there and the
  clock stops standing in for it.
- The weather sensors: the same receiver hears two of my neighbors' sensors,
  a different one from each kitchen window. That's the next post, after a week
  of comparing one against the forecast.
- An IR blaster for the Rowenta fan in the kitchen, maybe on ESPHome, which
  I've wanted to try.

---

*[How this was built](/how-i-work/): Claude Code wrote the Flipper remote app's
spec, and a cheaper model wrote the app from it, at my call. Claude Code also
wrote the firmware changes (the FIFO transmit, the USB logging fix, the
Ethernet build, the Vornado decoder), the Home Assistant buttons, helpers,
automations and dashboard, and the test scripts, including the Flipper checks
against the real remote and the broker-log analysis that found the dropouts. I
set the Flipper step's goal (every frame built from the decode) and ran the app
on the fan, which is what caught the oscillation labels. I picked the board and
the radio, set the constraint that the PoE module stays on, which decided the
pins, wired and soldered it, chose what the automations do and when, chose the
Flipper as the test vehicle because it was already proven against the fan, and
pushed back when Claude Code said one radio couldn't follow the remote and the
weather sensors at once. Tested: the Flipper app on the fan (Power, Timer,
Selection + and -, Lamp Mode and both oscillation codes), Power on and off
through the gateway, the transmit timing against the real remote, the dropout
fix, Ethernet and the Wi-Fi fallback, updates over the network, the real remote
next to the gateway, the leave-home rule on three real departures, the last one
checked against the fan itself, the bedtime rule once on a late arrival, and
the phone showing as home through eero. Not tested: what Fan Speeds, Airflow
Modes and Light Settings do, whether Selection + and - follow the last function
pressed, which way the head moves for the oscillation on/off code, the 10 PM
and 7 AM triggers, the hot rule, the leave rule since eero joined my presence,
the watch as a fallback for a dead phone, and PoE power.*
