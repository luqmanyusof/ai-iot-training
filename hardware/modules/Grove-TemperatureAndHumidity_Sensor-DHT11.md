---
title: Grove - Temperature & Humidity Sensor (DHT11)
description: Digital 1-wire temperature and humidity sensor — course spec card
part_type: sensor
interface: digital 1-wire
source: https://wiki.seeedstudio.com/Grove-TemperatureAndHumidity_Sensor/
card_status: COURSE SPEC CARD — written for this course. Not vendor content. Verify anything marked VERIFY on real hardware.
---

# Grove - Temperature & Humidity Sensor (DHT11)

> **Course-written spec card.** The vendor's full documentation is at the link above — this page
> carries only what this course needs, plus the ESP32-specific facts the vendor page does not cover.
> Attach this when the starter interview asks for a datasheet.

## What it is
A low-cost temperature and humidity sensor using a **single-wire digital protocol** — not I2C, not analog.
The kit uses the Crowtail DHT11, which is Grove-connector compatible.

## Specification

| Parameter | Value |
|---|---|
| Interface | **Digital 1-wire** (single data pin) |
| VCC | **3.3 - 5 V** — ESP32-safe |
| Measuring current | 1.3 - 2.1 mA |
| Average current | 0.5 - 1.1 mA |
| Humidity range | 20 - 90 % RH |
| **Sampling period** | **~1 Hz — poll no faster than every 1-2 s** |

## On the ROBO ESP32

- **Grove Port 3 — D26 or D25.** One signal wire; tell the AI which of the two you actually wired.
- It is **not** on the I2C bus.
- D25/D26 are ADC2-capable pins, which is irrelevant here because this sensor is read digitally — but it does mean the port is a poor choice for an analog sensor.

## Library

Seeed DHT library — https://github.com/Seeed-Studio/Grove_Temperature_And_Humidity_Sensor

## Gotchas

1. **It is NOT I2C.** If the AI reaches for `Wire` or address `0x38`, that is the **DHT20** — a
   different part. This is the headline trap of T3.
2. **The Seeed library defaults to `DHT 22`.** You must change the definition to `DHT 11` or your
   readings will be wrong or absent. The AI rarely mentions this.
3. **Do not poll faster than ~1 Hz.** Faster reads return stale or NaN values.
4. **A blocking read starves everything else.** The naive `delay(2000)` between reads freezes the
   display — use an independent `millis()` timer, later a FreeRTOS task.
5. **Handle NaN.** A failed read must keep the last good value and flag it, not propagate garbage.

## Course use

| Topic | Role |
|---|---|
| T3 | The environment input for the comfort monitor |
| T4-T9 | Carried forward as the primary sensor |
