---
title: Grove - I2C Hub
description: Passive I2C fan-out — OPTIONAL, not required as the course is scheduled
part_type: accessory
interface: I2C fan-out
source: https://wiki.seeedstudio.com/Grove-I2C_Hub/
card_status: COURSE SPEC CARD — written for this course. Not vendor content. Verify anything marked VERIFY on real hardware.
---

# Grove - I2C Hub

> **Course-written spec card.** The vendor's full documentation is at the link above — this page
> carries only what this course needs, plus the ESP32-specific facts the vendor page does not cover.
> Attach this when the starter interview asks for a datasheet.

> **Optional.** As the course is scheduled you do not need this: the OLED owns the single I2C Grove
> port for T3-T7, and the gesture sensor takes it from T8. **They are never required at the same time.**

## What it is
A passive fan-out that turns one Grove I2C port into several. It has **no I2C address of its own** —
it is wiring, not a device.

## Specification

| Parameter | Value |
|---|---|
| Interface | I2C fan-out (passive) |
| Operating voltage | 3.3 V / 5 V |
| Address | **none** — not an addressed device |

## When you would actually need it

The ROBO ESP32 has **exactly one I2C Grove port (Port 2, D21/D22)**. You need this hub (or a
Grove-to-QWIIC cable for the Maker port) **only if you want the OLED and the gesture sensor live
simultaneously** — for example a T9 mini project that wants both a local display and gesture input.

## Gotchas

1. **It does not solve addressing.** Two devices sharing an address still collide; a hub only shares wires.
2. **Bus capacitance adds up.** Long Grove cables plus a hub can degrade I2C at higher speeds.
3. **The Maker/QWIIC port is not a substitute without a cable** — it shares the same D21/D22 pins but
   uses a QWIIC/Stemma connector, so a Grove module needs a conversion cable.
