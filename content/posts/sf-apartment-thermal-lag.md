---
title: "Tracking Indoor vs Outdoor Temperature in an SF Apartment"
date: 2026-07-29
draft: false
tags: ["home-assistant", "zigbee", "home-lab", "data"]
description: "I bought an IKEA temp sensor for $8 while ordering batteries, stuck it in my living room, and ended up with a pretty clear picture of how SF apartments handle heat."
---

I needed AA and AAA batteries and grabbed an [IKEA TIMMERFLOTTE](https://www.ikea.com/us/en/p/timmerflotte-temperature-humidity-sensor-smart-50618957/) while I was at it. Zigbee temperature and humidity sensor, $8. I put it in my living room: middle of the apartment, between all the windows, so a reasonable representative reading. No real plan.

Ten days of data later, I have a clear picture of how this apartment moves heat — and it is not great.

## Setup

The TIMMERFLOTTE logs over Zigbee to Home Assistant. It reports on state change rather than on a fixed interval, so the data is dense: I get a reading whenever temp shifts by about 0.1°F. For outdoor data I'm using Pirate Weather, a privacy-respecting API that pulls from NOAA, exposed as a template sensor in HA:

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

Outdoor temperature swung from 53°F to 82°F over the 10-day window. Indoor moved from 65.5°F to 76.2°F.

That sounds fine until you look at when. The outdoor swing happens in hours. The indoor swing happens across days.

![Indoor vs outdoor temperature, Jul 22–25. Blue shading = ventilation window active.](/images/thermal-lag/chart_temp.png)

The shaded regions are when outdoor is cooler than indoor and above the 62°F floor — the window where opening the apartment actually helps. On a typical 67–70°F SF July day, that window is essentially all day. Outdoor rarely beats indoor until a hot spell hits.

July 21 was the outlier: 82°F peak at 4pm. Indoor tracked up to 76°F by evening — 6 degrees cooler than outside, but the apartment kept climbing for hours after the outdoor peak. The crossover back below indoor didn't happen until 7:20pm. By then the damage was done.

The overnight story is the telling one. Average outdoor low overnight (3–6am): 57.9°F. Average indoor: 68.7°F. Eleven degrees of heat the building is holding onto through the night while 58°F air sits right outside. It is a 4,000 square foot radiator.

## Humidity and dew point

![Dew point (top) and raw humidity (bottom), Jul 19–29.](/images/thermal-lag/chart_humidity.png)

This is where SF is actually fine. Indoor dew point averaged 56°F across 10 days and crossed the 60°F muggy threshold only 3% of the time. Outdoor humidity swings hard — up to 100% on fog mornings — but the apartment doesn't absorb it fast enough to matter. Ventilating pulls in cooler air without pumping in moisture.

## The fix

Open windows when outdoor is cooler than indoor, but not so cold you're just importing fog. I set 62°F as the floor.

I set up a Home Assistant automation that fires the moment that condition is met:

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

I also added a template sensor that tracks daily ventilation hours:

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

On a typical SF July day (high ~68°F), outdoor barely exceeds indoor at all. The ventilation window is effectively all day. On the July 21 heat spike, I had about 3 hours of actual lock-out before it cooled back down.

## Unplanned experiment: airflow changes

Two things changed during this window, in unknown order.

First, I flipped the fan direction: one fan blowing in through the kitchen, one blowing out through the bedroom. Cross-ventilation instead of two fans doing the same thing. I don't have the exact date for this one.

Second, on July 24 I added proper window screens to the bedroom and living room. Before that I had been opening windows but the screens were missing, so I was leaving them cracked rather than fully open. With screens in, I can leave windows wide open all day.

Both show up in the data.

![Daytime indoor–outdoor temperature delta, before and after airflow changes. Gray bar = Jul 21 heat spike (82°F outdoor high, excluded from averages).](/images/thermal-lag/chart_screens.png)

The gray bar is July 21: outdoor peaked at 82°F while indoor only reached 76°F, so the usual relationship inverted and I excluded it from the comparison. Every other day indoor was warmer than outdoor during the afternoon.

The pre-screen period (Jul 19–23) already shows a downward trend — July 20 is notably lower than July 19, and the recovery after the heat spike on Jul 22–23 doesn't fully return to the earlier baseline. That's likely the fan direction change taking effect. Then the screens pushed it further: Jul 25–28 averaged +2.2°F vs +5.9°F before, ending at 0.6°F on July 28.

I can't cleanly separate the two effects without knowing when the fans were switched. But the combined result is a 3.7°F reduction in the afternoon delta. The apartment that was stubbornly 6°F warmer than outside at midday is now tracking within 2°F on most days, and sometimes nearly equal.

## What's next

**Curtain timing.** My curtains open automatically at 6:45am. On a cool overcast morning there's no real reason to let the sun in early and add heat load. An adaptive version would delay opening on warm mornings to block solar gain longer, but only on weekends — I don't want my wake schedule varying on weekdays.

**Fan control.** I have two Rowenta fans. The table fan goes on a smart plug and can be automated directly. The tower fan uses an IR remote, so a smart plug can cut power but can't turn it on without an IR blaster. Once that's in place: fans on when the ventilation window opens, off when it closes or I'm not home.

---

*Data from Jul 19–29, 2026. Sensors: IKEA TIMMERFLOTTE (indoor, living room), Pirate Weather via Home Assistant (outdoor). San Francisco, CA.*
