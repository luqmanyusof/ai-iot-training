---
title: Grove - Rotary Angle Sensor
description: 300-degree 10k potentiometer, analog output — course spec card
part_type: analog input
interface: analog
source: https://wiki.seeedstudio.com/Grove-Rotary_Angle_Sensor/
card_status: COURSE SPEC CARD — written for this course. Not vendor content. Verify anything marked VERIFY on real hardware.
---

# Grove - Rotary Angle Sensor

> **Course-written spec card.** The vendor's full documentation is at the link above — this page
> carries only what this course needs, plus the ESP32-specific facts the vendor page does not cover.
> Attach this when the starter interview asks for a datasheet.

## What it is
A 10 k rotary potentiometer on a Grove connector. It outputs an analog voltage between 0 and VCC.
Seeed MPN **101020017**.

## Specification

| Parameter | Value |
|---|---|
| Interface | **Analog** (one ADC pin) |
| Rotation | **300 degrees**, linear |
| Resistance | 10 k ohm |
| Vendor-stated voltage | 4.75 - 5.0 - 5.25 V DC |
| Dimensions | 19 x 19 x 30.1 mm |

## On the ROBO ESP32 — read this before wiring

- **ADC1 Grove port (D32/D33) ONLY.** ADC2 stops working the moment WiFi is enabled.
- **Run it at 3.3 V, not the 5 V the vendor page states.** It is a passive resistive divider, so it
  works fine at 3.3 V — and it *must* be 3.3 V here, because the ESP32's analog input is rated
  0 - 3.3 V. Feeding it 5 V puts an out-of-range voltage on the ADC pin.
- No library needed: `analogRead()`.

## Gotchas

1. **ADC1 only.** The single most expensive mistake in this course — it works on Day 1 and returns
   garbage on Day 2 when the radio comes up. It looks exactly like a software bug.
2. **The vendor example code is wrong for ESP32, twice:**
   - it assumes `ADC_REF 5` — yours is **3.3**
   - it divides by **1023** (10-bit) — the ESP32 ADC is **12-bit, 0-4095**
   Copy that example unchanged and your angle calculation is wrong. The AI copies it unchanged.
3. **The ESP32 ADC is non-linear near the rails.** Expect compression near 0 and 4095; calibrate
   against the usable middle range.
4. **Raw reads jitter.** Filter (moving average) before the value drives anything visible.

## Course use

| Topic | Role |
|---|---|
| T2 | The analog/ADC lesson and the ADC1-vs-WiFi trap |
| T3-T6 | Comfort threshold setpoint |
| T7 | Your knob drives your partner's servo across the RS485 bus |
| T9 | Local setpoint / manual jog |
