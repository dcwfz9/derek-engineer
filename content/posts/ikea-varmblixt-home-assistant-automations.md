---
title: "IKEA Varmblixt as a Home Assistant Status Light"
date: 2026-04-13
draft: false
tags: ["home-assistant", "zigbee", "home-lab"]
description: "Turning the IKEA Varmblixt donut lamp into a context-aware status light with weather reactions, cycling alerts, door alerts, and three color scripts."
---

The IKEA Varmblixt is a $30 RGB+white Zigbee pendant. It pairs directly to ZHA in Home Assistant, has smooth color transitions, and a wide enough color gamut to be useful as a status indicator. I turned it into an ambient display for my apartment — weather during the day, a sunset script at night, a cycling green light at 6:30am, and a solid red door alarm when I forget to close the front door.

## Hardware

- **IKEA Varmblixt** ($30 at IKEA) — RGB+white, Zigbee
- Pairs over ZHA (no hub, no IKEA Dirigera needed)
- Entity: `light.led_light0x07c2_varmblixt_table_wall_lamp`

Pairing is not obvious. There's no reset button. To enter pairing mode, power-cycle the lamp 12 times in rapid succession — plug, unplug, plug, unplug... The lamp flashes white 3–4 times when it's ready. A smart plug makes this way easier than doing it by hand with the physical outlet. [Source](https://www.reddit.com/r/homeassistant/comments/1rnhn78/new_ikea_varmblixt/)

## Scripts

Three color scripts live in `scripts.yaml`. All use `mode: restart` so calling a new script immediately cancels the previous one without any cleanup logic.

### cotton_candy_sunset

The default sunset mode. Slow amber drift, 45-second crossfades between two warm tones. Runs continuously until midnight.

```yaml
cotton_candy_sunset:
  alias: Cotton Candy Sunset
  description: Slow dreamy fade between sunset amber and warm magenta — default sunset mode
  sequence:
  - repeat:
      count: 500
      sequence:
      - action: light.turn_on
        target:
          entity_id: light.led_light0x07c2_varmblixt_table_wall_lamp
        data:
          rgb_color: [220, 80, 5]
          brightness_pct: 50
          transition: 45
      - delay: 00:00:10
      - action: light.turn_on
        target:
          entity_id: light.led_light0x07c2_varmblixt_table_wall_lamp
        data:
          rgb_color: [255, 130, 10]
          brightness_pct: 50
          transition: 45
      - delay: 00:00:10
  mode: restart
```

### color_drift

Six colors, 4-second transitions, 5-second holds. Lava lamp pacing. Good for background ambience when not using the sunset mode.

```yaml
color_drift:
  alias: Color Drift
  description: Slow hypnotic color cycle on the donut — lava lamp vibe
  sequence:
  - repeat:
      count: 500
      sequence:
      - action: light.turn_on
        target:
          entity_id: light.led_light0x07c2_varmblixt_table_wall_lamp
        data: {rgb_color: [255, 134, 8], brightness_pct: 50, transition: 4}
      - delay: 00:00:05
      - action: light.turn_on
        target:
          entity_id: light.led_light0x07c2_varmblixt_table_wall_lamp
        data: {rgb_color: [200, 0, 180], brightness_pct: 50, transition: 4}
      - delay: 00:00:05
      - action: light.turn_on
        target:
          entity_id: light.led_light0x07c2_varmblixt_table_wall_lamp
        data: {rgb_color: [0, 180, 180], brightness_pct: 50, transition: 4}
      - delay: 00:00:05
      - action: light.turn_on
        target:
          entity_id: light.led_light0x07c2_varmblixt_table_wall_lamp
        data: {rgb_color: [120, 0, 220], brightness_pct: 50, transition: 4}
      - delay: 00:00:05
      - action: light.turn_on
        target:
          entity_id: light.led_light0x07c2_varmblixt_table_wall_lamp
        data: {rgb_color: [255, 60, 0], brightness_pct: 50, transition: 4}
      - delay: 00:00:05
      - action: light.turn_on
        target:
          entity_id: light.led_light0x07c2_varmblixt_table_wall_lamp
        data: {rgb_color: [0, 210, 40], brightness_pct: 50, transition: 4}
      - delay: 00:00:05
  mode: restart
```

### party_pulse

Five colors, 1-second transitions, 1-second holds. For when you want it to be annoying on purpose.

```yaml
party_pulse:
  alias: Party Pulse
  description: Fast energetic color flashes on the donut
  sequence:
  - repeat:
      count: 500
      sequence:
      - action: light.turn_on
        target:
          entity_id: light.led_light0x07c2_varmblixt_table_wall_lamp
        data: {rgb_color: [255, 0, 120], brightness_pct: 80, transition: 1}
      - delay: 00:00:01
      - action: light.turn_on
        target:
          entity_id: light.led_light0x07c2_varmblixt_table_wall_lamp
        data: {rgb_color: [50, 255, 0], brightness_pct: 80, transition: 1}
      - delay: 00:00:01
      - action: light.turn_on
        target:
          entity_id: light.led_light0x07c2_varmblixt_table_wall_lamp
        data: {rgb_color: [0, 220, 255], brightness_pct: 80, transition: 1}
      - delay: 00:00:01
      - action: light.turn_on
        target:
          entity_id: light.led_light0x07c2_varmblixt_table_wall_lamp
        data: {rgb_color: [255, 100, 0], brightness_pct: 80, transition: 1}
      - delay: 00:00:01
      - action: light.turn_on
        target:
          entity_id: light.led_light0x07c2_varmblixt_table_wall_lamp
        data: {rgb_color: [160, 0, 255], brightness_pct: 80, transition: 1}
      - delay: 00:00:01
  mode: restart
```

## Automations

### Weather + Sunset

Triggers on sunrise, sunset, and every hour. After sunset: runs `cotton_candy_sunset`. During the day: sets a static color based on the PirateWeather condition — fog is a cool blue-white, sun is yellow, rain is blue, clouds are light blue.

```yaml
- id: varmblixt_sf_weather_sunset
  alias: 'Varmblixt: SF Weather & Sunset'
  triggers:
  - trigger: sun
    event: sunset
  - trigger: sun
    event: sunrise
  - trigger: time_pattern
    hours: /1
  actions:
  - choose:
    - conditions:
      - condition: sun
        after: sunset
      sequence:
      - action: script.turn_on
        target:
          entity_id: script.cotton_candy_sunset
    - conditions:
      - condition: sun
        after: sunrise
        before: sunset
      sequence:
      - action: light.turn_on
        target:
          entity_id: light.led_light0x07c2_varmblixt_table_wall_lamp
        data:
          brightness_pct: 70
          rgb_color: >
            {% set s = states("weather.pirateweather") %}
            {% if s == "fog" %}[200,200,255]
            {% elif s == "sunny" %}[255,200,0]
            {% elif s in ["rainy","pouring"] %}[0,100,255]
            {% elif s in ["cloudy","partlycloudy"] %}[150,200,255]
            {% else %}[255,150,50]{% endif %}
  mode: single
```

| Condition | Color |
|-----------|-------|
| Sunny | Yellow `[255,200,0]` |
| Cloudy / Partly Cloudy | Light blue `[150,200,255]` |
| Rainy / Pouring | Blue `[0,100,255]` |
| Fog | Blue-white `[200,200,255]` |
| Other | Warm orange `[255,150,50]` |

### Cycling Green Light

At 6:30am, checks PirateWeather. If temperature is 55–78°F, wind under 15mph, and no rain or fog, the lamp goes green. It's a go/no-go signal before I check my phone. At 9am a second automation triggers the weather automation to revert it.

```yaml
- id: cycling_weather_green
  alias: Cycling Weather - Green Light
  triggers:
  - trigger: time
    at: 06:30:00
  conditions:
  - condition: template
    value_template: >
      {% set temp = state_attr('weather.pirateweather', 'temperature') | float(0) %}
      {% set wind = state_attr('weather.pirateweather', 'wind_speed') | float(99) %}
      {% set cond = states('weather.pirateweather') %}
      {{ temp >= 55 and temp <= 78
         and wind < 15
         and cond not in ['rainy', 'pouring', 'fog', 'hail', 'snowy',
                          'snowy-rainy', 'lightning', 'lightning-rainy', 'exceptional'] }}
  actions:
  - action: light.turn_on
    target:
      entity_id: light.led_light0x07c2_varmblixt_table_wall_lamp
    data:
      rgb_color: [0, 210, 40]
      brightness_pct: 50
  mode: single

- id: cycling_weather_revert
  alias: Cycling Weather - Revert at 9am
  triggers:
  - trigger: time
    at: 09:00:00
  actions:
  - action: automation.trigger
    target:
      entity_id: automation.varmblixt_sf_weather_sunset
  mode: single
```

### Door Open Alert

If the front door stays open for more than 10 seconds, the lamp goes solid red at 100% brightness — visible from anywhere in the apartment. When the door closes, it automatically reverts to the correct mode (sunset script or weather color depending on time of day).

```yaml
- id: door_open_light_flash
  alias: Alert - Door Open Light Red
  triggers:
  - trigger: state
    entity_id: binary_sensor.ikea_of_sweden_parasoll_door_window_sensor
    to: 'on'
    for:
      seconds: 10
  actions:
  - action: light.turn_on
    target:
      entity_id: light.led_light0x07c2_varmblixt_table_wall_lamp
    data:
      rgb_color: [255, 0, 0]
      brightness_pct: 100
  - wait_for_trigger:
    - trigger: state
      entity_id: binary_sensor.ikea_of_sweden_parasoll_door_window_sensor
      to: 'off'
    timeout:
      hours: 1
    continue_on_timeout: true
  - choose:
    - conditions:
      - condition: sun
        after: sunset
      sequence:
      - action: script.turn_on
        target:
          entity_id: script.cotton_candy_sunset
    - conditions:
      - condition: sun
        after: sunrise
        before: sunset
      sequence:
      - action: light.turn_on
        target:
          entity_id: light.led_light0x07c2_varmblixt_table_wall_lamp
        data:
          brightness_pct: 70
          rgb_color: >
            {% set s = states("weather.pirateweather") %}
            {% if s == "fog" %}[200,200,255]
            {% elif s == "sunny" %}[255,200,0]
            {% elif s in ["rainy","pouring"] %}[0,100,255]
            {% elif s in ["cloudy","partlycloudy"] %}[150,200,255]
            {% else %}[255,150,50]{% endif %}
  mode: single
```

### Midnight Off

Stops all running scripts and turns the lamp off at midnight. Without this, `cotton_candy_sunset` runs indefinitely on its 500-iteration loop.

```yaml
- id: '1758620017271'
  alias: Midnight
  triggers:
  - trigger: time
    at: 00:00:00
  actions:
  - action: script.turn_off
    target:
      entity_id:
      - script.cotton_candy_sunset
      - script.color_drift
      - script.party_pulse
  - action: light.turn_off
    target:
      entity_id:
      - light.all_lights
      - light.led_light0x07c2_varmblixt_table_wall_lamp
  mode: single
```

The explicit Varmblixt entity in the `light.turn_off` call is there because the lamp wasn't reliably included in `light.all_lights` before a full HA restart — turning off `all_lights` alone didn't always catch it.

## Notes

- **PirateWeather** is the weather integration used here. SF has brutal microclimate variance — Met.no (the HA default) was consistently wrong about fog and was useless for the cycling trigger. PirateWeather is a free Dark Sky API replacement that uses the same ML-based model and returns the same condition strings. Night and day difference for anything fog-related. Any weather integration that exposes a `weather.*` entity with standard condition states will work — just swap the entity ID.
- The `count: 500` in each script is a practical cap. At the slowest pace (cotton candy sunset: ~100 seconds per cycle) that's about 14 hours, more than enough to cover any night. The midnight automation cleans it up before the loop ends anyway.
- `mode: restart` on scripts means calling `script.turn_on` on one while another is running kills the previous immediately. No extra cleanup needed.
