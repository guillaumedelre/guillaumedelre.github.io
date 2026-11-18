---
title: "From a €10 sensor to a Home Assistant dashboard with a Raspberry Pi and MQTT"
aliases: ["/2019/11/17/from-a-10-sensor-to-a-home-assistant-dashboard-with-a-raspberry-pi-and-mqtt/"]
date: 2026-05-14
lastmod: 2026-11-18
tags: [python, raspberry-pi, home-assistant]
format: experiment
description: "A €10 BME280 sensor, a Raspberry Pi, and an MQTT broker: building a room climate monitor that feeds Home Assistant."
---

{{< lead >}}
The question was simple: what's the temperature and humidity in my home office right now? Not the weather outside, not a city average — the actual conditions in the room where I spend most of my day. Opening a weather app for that felt wrong.
{{< /lead >}}

A Raspberry Pi was already running on the shelf. A BME280 sensor costs around €10. This should have been a weekend project.

It mostly was, except for the part where I assumed reading a temperature sensor meant reading a register.

## Four wires and a chip

The Bosch BME280 measures temperature, humidity, and atmospheric pressure over I²C. Four wires to the Raspberry Pi GPIO pins, enable I²C in `raspi-config`, and the sensor shows up on the bus, at `0x77` on my board (`0x76` on many breakouts, depending on how the SDO pin is wired):

```bash
i2cdetect -y 1
```

That's the easy part. The catch is what happens next.

## You don't just read the temperature

The BME280 doesn't hand you `21.5°C`. It gives you raw ADC values: 20-bit integers for temperature and pressure, 16-bit for humidity, that mean nothing by themselves. To get an actual temperature, you have to:

1. Read the calibration coefficients Bosch burned into the chip's non-volatile memory at the factory (registers `0x88`, `0xA1`, `0xE1`)
2. Apply Bosch's compensation formulas, which use those coefficients to turn raw values into real measurements: integer arithmetic for temperature, double-precision floating point for pressure and humidity
3. Wait for the measurement to finish by polling the status register

The temperature compensation alone takes the raw value, applies a quadratic correction with three calibration constants, and spits out a value in hundredths of degrees Celsius. Pressure depends on the corrected temperature, and so does humidity.

It's all straight from the [Bosch datasheet][datasheet], nothing clever. I didn't start from scratch either: the driver began as Matt Hawkins' script from [Raspberry Pi Spy][rpi-spy], a direct port of those formulas. But it does mean the driver isn't a five-liner: it's a transcription of the spec.

## Getting the numbers off the Pi

Once the driver worked, the next question was how to get those values into Home Assistant. The simplest path: a Flask API with two endpoints.

`GET /bme280` returns the current reading as JSON. `GET /bme280/publish` reads the sensor and pushes the three values to an MQTT broker. A cron job on the Pi calls the publish endpoint every minute, and Home Assistant picks up the values as soon as they're published.

The MQTT discovery mechanism made the Home Assistant side almost frictionless. One `mosquitto_pub` command per sensor type — publishing a JSON config payload to the right topic — and the entities appear automatically in the UI. No `configuration.yaml` editing, no restart required.

{{< mermaid >}}
flowchart LR
  S[BME280] -- I²C --> D[bme280.py] --> A[Flask API] -- MQTT --> H[Home Assistant]
{{< /mermaid >}}

The full setup guide is [in the repo][repo].

## A note on the official Bosch SensorAPI

While working on the driver I peeked at the [official Bosch SensorAPI][sensorapi] for reference. Two things caught my attention.

For years, the Linux userspace example didn't work on a Raspberry Pi out of the box. Several contributors tripped over the same bug independently: `ioctl` was called before `dev_addr` was assigned, so the I²C device address never got set properly. The fix is obvious once you see it, and PRs documented it as early as 2021, but it only landed in May 2023. One of those older PRs is still open.

Then there's [PR #94][pr94] (opened in January 2025 and still open), reporting undefined behavior in `bme280_get_sensor_mode()`: the left operand of a bitwise `&` is an uninitialized variable, caught by static analysis.

The chip itself is great. But manufacturer reference code is a starting point, not gospel. Working through the compensation algorithm line by line against the datasheet meant I understood every line of it. When a reading looks weird, there's no mystery C library to blame.

## What I didn't expect

The Bosch calibration is non-negotiable. I started by reading the raw temperature register directly and scaling it naively. The result was numbers that looked almost plausible and were completely wrong. Without the compensation algorithm, the output means nothing.

Polling suited this project better than events: the sensor doesn't push data, you ask it for a reading. A cron job every minute is all room monitoring needs. Sampling much faster would be overkill, and it would warm the chip up enough to skew the temperature.

As for MQTT discovery, I think it's underrated. Manually declaring sensors in `configuration.yaml` works, but auto-discovery feels right: publish a config payload once, and Home Assistant takes it from there. Adding a new sensor type later takes about thirty seconds.

The room is now 21.4°C and 47% humidity. I know this without opening anything.

{{< github repo="guillaumedelre/bme280" showThumbnail=true >}}

[datasheet]: https://www.bosch-sensortec.com/media/boschsensortec/downloads/datasheets/bst-bme280-ds002.pdf
[rpi-spy]: https://www.raspberrypi-spy.co.uk/
[repo]: https://github.com/guillaumedelre/bme280
[sensorapi]: https://github.com/boschsensortec/BME280_SensorAPI
[pr94]: https://github.com/boschsensortec/BME280_SensorAPI/pull/94
