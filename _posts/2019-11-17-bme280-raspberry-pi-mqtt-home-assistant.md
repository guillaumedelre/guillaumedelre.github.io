---
layout: post
title: "From a €10 sensor to a Home Assistant dashboard with a Raspberry Pi and MQTT"
date: 2019-11-17
categories: [iot]
tags: [raspberry-pi, bme280, mqtt, flask, i2c]
description: "A €10 BME280 sensor, a Raspberry Pi, and an MQTT broker: building a room climate monitor that feeds Home Assistant."
---

The question was simple: what's the temperature and humidity in my home office right now? Not the weather outside, not a city average — the actual conditions in the room where I spend most of my day. Opening a weather app for that felt wrong.

A Raspberry Pi was already running on the shelf. A BME280 sensor costs around €10. This should have been a weekend project.

It mostly was, except for the part where I assumed reading a temperature sensor meant reading a register.

## Four wires and a chip

The Bosch BME280 measures temperature, humidity, and atmospheric pressure over I²C. Four wires to the Raspberry Pi GPIO pins, enable I²C in `raspi-config`, and the sensor shows up at address `0x77` on the bus:

```bash
i2cdetect -y 1
```

That's the easy part. The catch is what happens next.

## You don't just read the temperature

The BME280 doesn't hand you `21.5°C`. It gives you raw ADC values: 20-bit integers that mean absolutely nothing by themselves. To get an actual temperature, you have to:

1. Read the calibration coefficients Bosch burned into the chip's EEPROM at the factory (registers `0x88`, `0xA1`, `0xE1`)
2. Apply Bosch's compensation formulas: double-precision floating point arithmetic that uses those coefficients to turn raw values into real measurements
3. Wait for the measurement to finish by polling the status register

The temperature compensation alone takes the raw value, applies a quadratic correction with three calibration constants, and spits out a value in hundredths of degrees Celsius. Pressure depends on the corrected temperature. Humidity depends on both.

It's all straight from the <a href="https://www.bosch-sensortec.com/media/boschsensortec/downloads/datasheets/bst-bme280-ds002.pdf" target="_blank" rel="noopener noreferrer">Bosch datasheet</a>, nothing clever. But it does mean the driver isn't a five-liner. It's implementing a spec, not importing a library.

## Making it network-accessible

Once the driver worked, the next question was how to get those values into Home Assistant. The simplest path: a Flask API with two endpoints.

`GET /bme280` returns the current reading as JSON. `GET /bme280/publish` reads the sensor and pushes the three values to an MQTT broker. A cron job on the Pi calls the publish endpoint every few minutes, and Home Assistant picks up the values in real time.

The MQTT discovery mechanism made the Home Assistant side almost frictionless. One `mosquitto_pub` command per sensor type — publishing a JSON config payload to the right topic — and the entities appear automatically in the UI. No `configuration.yaml` editing, no restart required.

```
BME280  ──I²C──►  bme280.py  ──►  Flask API  ──MQTT──►  Home Assistant
```

The full setup guide is <a href="https://github.com/guillaumedelre/bme280" target="_blank" rel="noopener noreferrer">in the repo</a>.

## What I didn't expect

:thermometer: **The Bosch calibration is non-negotiable.** I started by reading the raw temperature register directly and scaling it naively. The result was numbers that looked almost plausible and were completely wrong. The compensation algorithm isn't optional decoration, it's what makes the output mean anything.

:clock1: **Polling beats events here.** The sensor doesn't push data, you ask it for a reading. A cron job every minute is all you need for room monitoring. Real-time streaming would be overkill and would probably wear out the sensor faster.

:house: **MQTT discovery is underrated.** Manually declaring sensors in `configuration.yaml` works, but auto-discovery just feels right. Publish a config payload once, and Home Assistant takes it from there. Adding a new sensor type later takes about thirty seconds.

The room is now 21.4°C and 47% humidity. I know this without opening anything.

## A note on the official Bosch SensorAPI

While writing the driver I peeked at the <a href="https://github.com/boschsensortec/BME280_SensorAPI" target="_blank" rel="noopener noreferrer">official Bosch SensorAPI</a> for reference. Two things caught my attention.

The Linux userspace example doesn't actually work on a Raspberry Pi out of the box. Several contributors tripped over the same bug independently: `ioctl` is called before `dev_addr` is assigned, so the I²C device address never gets set properly. The fix is obvious once you see it, and multiple PRs documented it, but they sat open for years. Some still are.

Then there's <a href="https://github.com/boschsensortec/BME280_SensorAPI/pull/94" target="_blank" rel="noopener noreferrer">PR #94</a> (still open as of early 2025), reporting undefined behavior in `bme280_get_sensor_mode()`: the left operand of a bitwise `&` is an uninitialized variable, caught by static analysis.

The chip itself is great. But manufacturer reference code is a starting point, not gospel. Implementing the compensation algorithm straight from the datasheet meant I understood every line of it. When a reading looks weird, there's no mystery C library to blame.

<div style="border: 1px solid #e8e8e8; padding: 16px; margin-top: 2em; border-radius: 3px;">
  <img src="https://cdn.simpleicons.org/github" width="20" style="vertical-align: middle; margin-right: 8px;" />
  <strong><a href="https://github.com/guillaumedelre/bme280" target="_blank" rel="noopener noreferrer">guillaumedelre/bme280</a></strong>
  <p style="margin: 8px 0 0; color: #828282; font-size: 14px;">Python driver for the BME280 sensor — temperature, humidity, and pressure over I²C, with MQTT publishing and Home Assistant integration.</p>
</div>

