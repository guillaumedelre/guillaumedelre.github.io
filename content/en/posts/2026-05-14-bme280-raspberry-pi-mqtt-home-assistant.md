---
title: "From a €10 sensor to a Home Assistant dashboard with a Raspberry Pi and MQTT"
aliases: ["/2019/11/17/from-a-10-sensor-to-a-home-assistant-dashboard-with-a-raspberry-pi-and-mqtt/"]
date: 2026-05-14
lastmod: 2026-11-17
tags: [python, raspberry-pi, home-assistant]
format: experiment
description: "A €10 BME280 sensor, a Raspberry Pi, and an MQTT broker: building a room climate monitor that feeds Home Assistant."
---

The question was simple: what's the temperature and humidity in my home office right now? Not the weather outside, not a city average — the actual conditions in the room where I spend most of my day. Opening a weather app for that felt wrong.

A Raspberry Pi was already running on the shelf. A BME280 sensor costs around €10. This should have been a weekend project.

It mostly was, except for the part where I assumed reading a temperature sensor meant reading a register.

## Four wires and a chip

The Bosch BME280 measures temperature, humidity, and atmospheric pressure over I²C. Four wires to the Raspberry Pi GPIO pins, enable I²C in `raspi-config`, and the sensor shows up on the bus, at `0x77` on my board (`0x76` on many breakouts, depending on how the SDO pin is wired):

```bash
i2cdetect -y 1
```

That's the easy part. The catch is what happens next.

## You don't just read the temperature

The BME280 doesn't hand you `21.5°C`. It gives you raw ADC values: 20-bit integers for temperature and pressure, 16-bit for humidity, that mean absolutely nothing by themselves. To get an actual temperature, you have to:

1. Read the calibration coefficients Bosch burned into the chip's non-volatile memory at the factory (registers `0x88`, `0xA1`, `0xE1`)
2. Apply Bosch's compensation formulas, which use those coefficients to turn raw values into real measurements: integer arithmetic for temperature, double-precision floating point for pressure and humidity
3. Wait for the measurement to finish by polling the status register

The temperature compensation alone takes the raw value, applies a quadratic correction with three calibration constants, and spits out a value in hundredths of degrees Celsius. Pressure depends on the corrected temperature. Humidity depends on both.

It's all straight from the <a href="https://www.bosch-sensortec.com/media/boschsensortec/downloads/datasheets/bst-bme280-ds002.pdf" target="_blank" rel="noopener noreferrer">Bosch datasheet</a>, nothing clever. I didn't start from scratch either: the driver began as Matt Hawkins' script from <a href="https://www.raspberrypi-spy.co.uk/" target="_blank" rel="noopener noreferrer">Raspberry Pi Spy</a>, a direct port of those formulas. But it does mean the driver isn't a five-liner. It's implementing a spec, not importing a library.

## Making it network-accessible

Once the driver worked, the next question was how to get those values into Home Assistant. The simplest path: a Flask API with two endpoints.

`GET /bme280` returns the current reading as JSON. `GET /bme280/publish` reads the sensor and pushes the three values to an MQTT broker. A cron job on the Pi calls the publish endpoint every minute, and Home Assistant picks up the values as soon as they're published.

The MQTT discovery mechanism made the Home Assistant side almost frictionless. One `mosquitto_pub` command per sensor type — publishing a JSON config payload to the right topic — and the entities appear automatically in the UI. No `configuration.yaml` editing, no restart required.

```
BME280  ──I²C──►  bme280.py  ──►  Flask API  ──MQTT──►  Home Assistant
```

The full setup guide is <a href="https://github.com/guillaumedelre/bme280" target="_blank" rel="noopener noreferrer">in the repo</a>.

## What I didn't expect

**The Bosch calibration is non-negotiable.** I started by reading the raw temperature register directly and scaling it naively. The result was numbers that looked almost plausible and were completely wrong. The compensation algorithm isn't optional decoration, it's what makes the output mean anything.

**Polling beats events here.** The sensor doesn't push data, you ask it for a reading. A cron job every minute is all you need for room monitoring. Sampling much faster would be overkill, and it would warm the chip up enough to skew the temperature.

**MQTT discovery is underrated.** Manually declaring sensors in `configuration.yaml` works, but auto-discovery just feels right. Publish a config payload once, and Home Assistant takes it from there. Adding a new sensor type later takes about thirty seconds.

The room is now 21.4°C and 47% humidity. I know this without opening anything.

## A note on the official Bosch SensorAPI

While working on the driver I peeked at the <a href="https://github.com/boschsensortec/BME280_SensorAPI" target="_blank" rel="noopener noreferrer">official Bosch SensorAPI</a> for reference. Two things caught my attention.

For years, the Linux userspace example didn't work on a Raspberry Pi out of the box. Several contributors tripped over the same bug independently: `ioctl` was called before `dev_addr` was assigned, so the I²C device address never got set properly. The fix is obvious once you see it, and PRs documented it as early as 2021, but it only landed in May 2023. One of those older PRs is still open.

Then there's <a href="https://github.com/boschsensortec/BME280_SensorAPI/pull/94" target="_blank" rel="noopener noreferrer">PR #94</a> (opened in January 2025 and still open), reporting undefined behavior in `bme280_get_sensor_mode()`: the left operand of a bitwise `&` is an uninitialized variable, caught by static analysis.

The chip itself is great. But manufacturer reference code is a starting point, not gospel. Working through the compensation algorithm line by line against the datasheet meant I understood every line of it. When a reading looks weird, there's no mystery C library to blame.

<a class="gh-card" href="https://github.com/guillaumedelre/bme280" target="_blank" rel="noopener noreferrer">
  <span class="gh-card__head">
    <svg width="20" height="20" viewBox="0 0 24 24" aria-hidden="true"><path d="M12 .297c-6.63 0-12 5.373-12 12 0 5.303 3.438 9.8 8.205 11.385.6.113.82-.258.82-.577 0-.285-.01-1.04-.015-2.04-3.338.724-4.042-1.61-4.042-1.61C4.422 18.07 3.633 17.7 3.633 17.7c-1.087-.744.084-.729.084-.729 1.205.084 1.838 1.236 1.838 1.236 1.07 1.835 2.809 1.305 3.495.998.108-.776.417-1.305.76-1.605-2.665-.3-5.466-1.332-5.466-5.93 0-1.31.465-2.38 1.235-3.22-.135-.303-.54-1.523.105-3.176 0 0 1.005-.322 3.3 1.23.96-.267 1.98-.399 3-.405 1.02.006 2.04.138 3 .405 2.28-1.552 3.285-1.23 3.285-1.23.645 1.653.24 2.873.12 3.176.765.84 1.23 1.91 1.23 3.22 0 4.61-2.805 5.625-5.475 5.92.42.36.81 1.096.81 2.22 0 1.606-.015 2.896-.015 3.286 0 .315.21.69.825.57C20.565 22.092 24 17.592 24 12.297c0-6.627-5.373-12-12-12"/></svg>
    <span class="gh-card__repo">guillaumedelre/bme280</span>
  </span>
  <span class="gh-card__desc">Python driver for the BME280 sensor — temperature, humidity, and pressure over I²C, with MQTT publishing and Home Assistant integration.</span>
</a>

