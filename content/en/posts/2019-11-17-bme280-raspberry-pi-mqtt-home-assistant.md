---
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

**The Bosch calibration is non-negotiable.** I started by reading the raw temperature register directly and scaling it naively. The result was numbers that looked almost plausible and were completely wrong. The compensation algorithm isn't optional decoration, it's what makes the output mean anything.

**Polling beats events here.** The sensor doesn't push data, you ask it for a reading. A cron job every minute is all you need for room monitoring. Real-time streaming would be overkill and would probably wear out the sensor faster.

**MQTT discovery is underrated.** Manually declaring sensors in `configuration.yaml` works, but auto-discovery just feels right. Publish a config payload once, and Home Assistant takes it from there. Adding a new sensor type later takes about thirty seconds.

The room is now 21.4°C and 47% humidity. I know this without opening anything.

## A note on the official Bosch SensorAPI

While writing the driver I peeked at the <a href="https://github.com/boschsensortec/BME280_SensorAPI" target="_blank" rel="noopener noreferrer">official Bosch SensorAPI</a> for reference. Two things caught my attention.

The Linux userspace example doesn't actually work on a Raspberry Pi out of the box. Several contributors tripped over the same bug independently: `ioctl` is called before `dev_addr` is assigned, so the I²C device address never gets set properly. The fix is obvious once you see it, and multiple PRs documented it, but they sat open for years. Some still are.

Then there's <a href="https://github.com/boschsensortec/BME280_SensorAPI/pull/94" target="_blank" rel="noopener noreferrer">PR #94</a> (still open as of early 2025), reporting undefined behavior in `bme280_get_sensor_mode()`: the left operand of a bitwise `&` is an uninitialized variable, caught by static analysis.

The chip itself is great. But manufacturer reference code is a starting point, not gospel. Implementing the compensation algorithm straight from the datasheet meant I understood every line of it. When a reading looks weird, there's no mystery C library to blame.

<style>
.gh-card {
  display: block;
  border: 1px solid #d0d7de;
  padding: 16px;
  margin-top: 2em;
  border-radius: 6px;
  text-decoration: none !important;
}
.gh-card:hover { border-color: #8c959f; }
.gh-card,
.gh-card *,
.md-content .gh-card,
.md-content .gh-card * { text-decoration: none !important; }
.gh-card__head { display: flex; align-items: center; gap: 8px; }
.gh-card__head svg { flex-shrink: 0; fill: #1f2328 !important; }
.gh-card__repo { font-weight: 600; color: #1f2328 !important; }
.gh-card__desc { margin: 8px 0 0; color: #59636e !important; font-size: 14px; }
[data-theme=dark] .gh-card { border-color: #30363d; }
[data-theme=dark] .gh-card:hover { border-color: #6e7681; }
[data-theme=dark] .gh-card__head svg { fill: #e6edf3 !important; }
[data-theme=dark] .gh-card__repo { color: #e6edf3 !important; }
[data-theme=dark] .gh-card__desc { color: #8b949e !important; }
</style>
<a class="gh-card" href="https://github.com/guillaumedelre/bme280" target="_blank" rel="noopener noreferrer">
  <span class="gh-card__head">
    <svg width="20" height="20" viewBox="0 0 24 24" aria-hidden="true"><path d="M12 .297c-6.63 0-12 5.373-12 12 0 5.303 3.438 9.8 8.205 11.385.6.113.82-.258.82-.577 0-.285-.01-1.04-.015-2.04-3.338.724-4.042-1.61-4.042-1.61C4.422 18.07 3.633 17.7 3.633 17.7c-1.087-.744.084-.729.084-.729 1.205.084 1.838 1.236 1.838 1.236 1.07 1.835 2.809 1.305 3.495.998.108-.776.417-1.305.76-1.605-2.665-.3-5.466-1.332-5.466-5.93 0-1.31.465-2.38 1.235-3.22-.135-.303-.54-1.523.105-3.176 0 0 1.005-.322 3.3 1.23.96-.267 1.98-.399 3-.405 1.02.006 2.04.138 3 .405 2.28-1.552 3.285-1.23 3.285-1.23.645 1.653.24 2.873.12 3.176.765.84 1.23 1.91 1.23 3.22 0 4.61-2.805 5.625-5.475 5.92.42.36.81 1.096.81 2.22 0 1.606-.015 2.896-.015 3.286 0 .315.21.69.825.57C20.565 22.092 24 17.592 24 12.297c0-6.627-5.373-12-12-12"/></svg>
    <span class="gh-card__repo">guillaumedelre/bme280</span>
  </span>
  <span class="gh-card__desc">Python driver for the BME280 sensor — temperature, humidity, and pressure over I²C, with MQTT publishing and Home Assistant integration.</span>
</a>

