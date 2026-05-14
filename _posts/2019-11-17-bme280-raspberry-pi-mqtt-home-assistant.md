---
layout: post
title: "From a €10 sensor to a Home Assistant dashboard with a Raspberry Pi and MQTT"
date: 2019-11-17
categories: [python, iot, home-assistant]
tags: [raspberry-pi, bme280, mqtt, flask, i2c]
---

The question was simple: what's the temperature and humidity in my home office right now? Not the weather outside, not a city average — the actual conditions in the room where I spend most of my day. Opening a weather app for that felt wrong.

A Raspberry Pi was already running on the shelf. A BME280 sensor costs around €10. This should have been a weekend project.

It mostly was, except for the part where I assumed reading a temperature sensor meant reading a register.

## :electric_plug: Four wires and a chip

The Bosch BME280 measures temperature, humidity, and atmospheric pressure over I²C. Four wires to the Raspberry Pi GPIO pins, enable I²C in `raspi-config`, and the sensor shows up at address `0x77` on the bus:

```bash
i2cdetect -y 1
```

That's the easy part. The catch is what happens next.

## :abacus: You don't just read the temperature

The BME280 doesn't give you `21.5°C` directly. It gives you raw ADC values — 20-bit integers with no physical meaning by themselves. To get an actual temperature, you have to:

1. Read the calibration coefficients that Bosch burned into the chip's EEPROM at the factory (registers `0x88`, `0xA1`, `0xE1`)
2. Apply Bosch's compensation formulas — double-precision floating point arithmetic that uses those coefficients to convert raw values into real measurements
3. Wait for the measurement to complete by polling the status register

The compensation algorithm for temperature alone takes the raw value, applies a quadratic correction using three calibration constants, and produces a value in hundredths of degrees Celsius. Pressure depends on the corrected temperature. Humidity depends on both.

None of this is guesswork — it's straight from the [Bosch datasheet][datasheet]. But it means the driver isn't trivial. It's not a library import; it's implementing a spec.

## :globe_with_meridians: Making it network-accessible

Once the driver worked, the next question was how to get those values into Home Assistant. The simplest path: a Flask API with two endpoints.

`GET /bme280` returns the current reading as JSON. `GET /bme280/publish` reads the sensor and pushes the three values to an MQTT broker. A cron job on the Pi calls the publish endpoint every few minutes, and Home Assistant picks up the values in real time.

The MQTT discovery mechanism made the Home Assistant side almost frictionless. One `mosquitto_pub` command per sensor type — publishing a JSON config payload to the right topic — and the entities appear automatically in the UI. No `configuration.yaml` editing, no restart required.

```
BME280  ──I²C──►  bme280.py  ──►  Flask API  ──MQTT──►  Home Assistant
```

The full setup guide is [in the repo][repo].

## :bulb: What I didn't expect

:thermometer: **The Bosch calibration is non-negotiable.** I initially tried reading the raw temperature register directly and scaling it naively. The result was plausible-looking garbage. The compensation algorithm exists because the sensor's raw output is meaningless without it.

:clock1: **Polling beats events here.** The sensor doesn't push data — you ask it for a reading. A cron job every minute is all you need for environmental monitoring. Real-time streaming would be overkill and would wear out the sensor faster.

:house: **MQTT discovery is underrated.** Manually declaring sensors in `configuration.yaml` works, but discovering them automatically feels like the right abstraction. Publish a config payload once, and Home Assistant handles the rest. Adding a new sensor type later takes thirty seconds.

The room is now 21.4°C and 47% humidity. I know this without opening anything.

## :warning: A note on the official Bosch SensorAPI

While writing the driver I looked at the [official Bosch SensorAPI][bosch-api] for reference. Two things stood out.

First, the Linux userspace example in the repo doesn't work on a Raspberry Pi as shipped. Several contributors independently discovered the same bug: `ioctl` is called before `dev_addr` is assigned, so the I²C device address is never properly set. The fix is straightforward, and multiple PRs documented it — but they sat open for years before corrected versions were eventually merged. The original PRs are still open.

Second, as of early 2025, [PR #94][pr94] reports undefined behavior in `bme280_get_sensor_mode()`: the left operand of a bitwise `&` is an uninitialized variable, caught by static analysis. It's still open.

None of this is a reason to avoid the chip — the hardware is excellent. But it's a reminder that reference code from a manufacturer is a starting point, not ground truth. Implementing the compensation algorithm from the datasheet directly, rather than wrapping the official C library, meant I understood every line of it. When something reads wrong, there's no black box to blame.

[datasheet]: https://www.bosch-sensortec.com/media/boschsensortec/downloads/datasheets/bst-bme280-ds002.pdf
[repo]: https://github.com/guillaumedelre/bme280
[bosch-api]: https://github.com/boschsensortec/BME280_SensorAPI
[pr94]: https://github.com/boschsensortec/BME280_SensorAPI/pull/94
