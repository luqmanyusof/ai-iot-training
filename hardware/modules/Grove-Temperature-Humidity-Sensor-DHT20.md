---
title: Grove - Temperature & Humidity Sensor (DHT20)
description: I2C temp/humidity sensor — NOT in the kit; kept as evidence for the T3 trap
part_type: sensor (reference only)
interface: I2C
source: https://wiki.seeedstudio.com/Grove-Temperature-Humidity-Sensor-DH20/
card_status: COURSE SPEC CARD — written for this course. Not vendor content. Verify anything marked VERIFY on real hardware.
---

# Grove - Temperature & Humidity Sensor (DHT20)

> **Course-written spec card.** The vendor's full documentation is at the link above — this page
> carries only what this course needs, plus the ESP32-specific facts the vendor page does not cover.
> Attach this when the starter interview asks for a datasheet.

> **This part is NOT in your kit.** It is documented here for one reason: it is the part the AI
> confuses your DHT11 with. When the AI writes I2C code at address `0x38` for your sensor, this
> page is the evidence that it has reached for the wrong device.

## What it is
The I2C successor to the DHT11 — higher accuracy, wider range, and a **completely different interface**.

## Specification

| Parameter | DHT20 (this part) | DHT11 (your kit) |
|---|---|---|
| **Interface** | **I2C, address `0x38`** | **digital 1-wire** |
| Input voltage | 2.0 - 5.5 V | 3.3 - 5 V |
| Temperature range | -40 to +80 C, +/-0.5 C | limited, +/-2 C |
| Humidity range | 0 - 100 % RH, +/-3 % RH | 20 - 90 % RH |

## Why this card exists

The T3 trap in one line: **`0x38` is this sensor, not yours.** If the AI produces `Wire.begin()`
and an address for your temperature sensor, print this table next to its output — that contrast is
the caught-AI story, verified against documentation.
