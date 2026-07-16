---
title: "Tracking Indoor vs Outdoor Temperature in an SF Apartment"
date: 2026-07-15
draft: true
tags: ["home-assistant", "zigbee", "home-lab", "data"]
description: "I bought an IKEA temp sensor for $8 while ordering batteries, stuck it in my living room, and ended up with a pretty clear picture of how SF apartments handle heat."
---

I needed AA and AAA batteries and grabbed an [IKEA TIMMERFLOTTE](https://www.ikea.com/us/en/p/timmerflotte-temperature-humidity-sensor-smart-50618957/) while I was at it. It's a Zigbee temperature and humidity sensor, $8. I put it in my living room, which is the middle of the apartment, between all the windows, so it seemed like a decent spot for representative data. No real plan.

The sensor started logging on July 13, which turned out to be one of those rare days where SF briefly hits 80°F. Typical highs here in early July are 63-65°F, so it was about 15°F above normal for two days before dropping back down. Lucky timing for the experiment.

## Setup

The TIMMERFLOTTE logs over Zigbee to Home Assistant. It reports on state change rather than on a fixed interval, so the data is dense: I get a reading whenever the temp shifts by about 0.1°F. For outdoor data I'm using Pirate Weather, a privacy-respecting weather API that pulls from NOAA, exposed as a template sensor in HA.

```yaml
template:
  - sensor:
      - name: "Outdoor Temperature"
        state: "{{ state_attr('weather.pirateweather', 'temperature') }}"
        unit_of_measurement: "°F"
        device_class: temperature
        state_class: measurement
```

## What the data shows

Three days overlaid, indoor solid and outdoor dashed:

*[chart: 3-day overlay]*

The apartment barely cools overnight. Monday outdoor dropped to 58°F by 2am. Indoor at that same time: 70°F. 12 degrees of heat that the building held onto through the night while perfectly cool air was sitting right outside. Same pattern every day, outdoor swings 20°F between night and afternoon, indoor moves maybe 5°F.

There's also a crossover that happens every afternoon around 2-3pm, when outdoor finally drops back below indoor after climbing all morning. Monday that was 2:54pm. Tuesday 1:48pm. Wednesday was a cooler day (high ~68°F) and outdoor never exceeded indoor at all.

The dew point picture is less alarming. Indoor is sitting in the 55-60°F range, right at the comfortable threshold. SF's marine layer keeps outdoor dew point surprisingly low even in July, so ventilating actually helps rather than pumping in humid air:

*[chart: dew point + humidity]*

## Why this happens

SF apartments get solar gain all afternoon, the concrete and drywall absorb it, and then overnight the building releases that heat back into the room slowly instead of letting it escape outside. The insulation that keeps you warm in winter works against you in summer.

## The fix

Open windows when outdoor is cooler than indoor, but not so cold you're just pumping in fog. I set 62°F as the floor.

I set up a Home Assistant automation that fires a notification the moment that condition is met, any time of day:

```yaml
- id: window_ventilation_alert
  alias: Window Ventilation Alert
  mode: single
  triggers:
    - trigger: numeric_state
      entity_id: sensor.outdoor_temperature
      below: sensor.living_room_timmerflotte_temp_hmd_sensor_temperature
  conditions:
    - condition: numeric_state
      entity_id: sensor.outdoor_temperature
      above: 62
  actions:
    - action: notify.mobile_app_dereks_iphone
      data:
        title: "Open windows now"
        message: >
          Outdoor ({{ states('sensor.outdoor_temperature') | round(0) }}°F)
          just dropped below indoor
          ({{ states('sensor.living_room_timmerflotte_temp_hmd_sensor_temperature') | round(0) }}°F).
```

I also added two HA sensors to track the ventilation opportunity over time:

```yaml
template:
  - binary_sensor:
      - name: "Ventilation Opportunity"
        state: >
          {{ states('sensor.outdoor_temperature') | float(0) <
             states('sensor.living_room_timmerflotte_temp_hmd_sensor_temperature') | float(0)
             and states('sensor.outdoor_temperature') | float(0) > 62 }}
        device_class: window

sensor:
  - platform: history_stats
    name: "Ventilation Hours Today"
    entity_id: binary_sensor.ventilation_opportunity
    state: "on"
    type: time
    start: "{{ now().replace(hour=0, minute=0, second=0, microsecond=0) }}"
    end: "{{ now() }}"
```

Tuesday (high ~77°F) gave 6 hours of ventilation window. Monday was 4h50m before outdoor dropped below 62°F.

## What's next

A few things I want to build out once I have more data:

**Curtain timing.** My curtains open automatically at 6:45am, which is also a good detail for a future post. On a cool overcast morning there's no real reason to let the sun in early and add heat load. An adaptive version would adjust the open time based on forecast and current indoor temp, but on weekends only since I don't want my wake time varying on weekdays.

**Fan control.** I have two Rowenta fans. The table fan goes on a smart plug and can be automated directly based on the ventilation score. The tower fan uses an IR remote, so a smart plug can cut power but can't wake it without an IR blaster. Either waiting for the IR Mate to come back in stock, or building something with an ESP32 and an IR LED, which I've wanted to do anyway. Once that's in place: fans on when the ventilation window opens, off when it closes or when I leave.

**Ten days of data.** The sensor started logging July 13. I'll revisit with a full dataset once I have it, probably around July 23.

---

*Data from Jul 13-15, 2026. Sensors: IKEA TIMMERFLOTTE (indoor, living room), Pirate Weather via Home Assistant (outdoor). San Francisco, CA.*
