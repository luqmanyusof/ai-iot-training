---
title: Grove Smart IR Gesture Sensor (PAJ7660)
description: AI gesture recognition over I2C — course spec card
part_type: sensor
interface: I2C (or SPI)
source: https://wiki.seeedstudio.com/grove_gesture_paj7660/
card_status: COURSE SPEC CARD — written for this course. Not vendor content. Verify anything marked VERIFY on real hardware.
---

# Grove Smart IR Gesture Sensor (PAJ7660)

> **Course-written spec card.** The vendor's full documentation is at the link above — this page
> carries only what this course needs, plus the ESP32-specific facts the vendor page does not cover.
> Attach this when the starter interview asks for a datasheet.

## What it is
An infrared camera sensor with an on-board gesture-recognition algorithm. It reports **recognised
gestures**, not raw pixels — push, pinch, tap, grab, rotation, thumb up/down, static and more
(15+ documented).

## Specification

| Parameter | Value |
|---|---|
| Interface | **I2C** (SPI also supported, for image output) |
| **I2C address** | **`0x73` — VERIFY by bus scan.** The vendor page itself is not definitive |
| Detection range | **5 - 40 cm** (vendor-stated) |
| Gestures | 15+ |
| Board size | 4.3 x 2.1 cm |
| Connectors | Grove, Type-C, Seeed Studio XIAO |

## On the ROBO ESP32

- **Grove Port 2 (D21/D22)** — the board's only I2C Grove port.
- Takes that port from **T8 onward**; the OLED comes off and the device runs headless.
- Library: Seeed PAJ7660 Arduino library.

## Gotchas

1. **Scan for the address — do not accept the AI's answer.** It will state an I2C address with total
   confidence. The vendor documentation itself flags `0x73` as needing verification.
2. **Detection range matters.** Too close or too far and gestures simply do not register. The vendor
   figure is **5-40 cm** — confirm the usable window on your own bench.
3. **Plain background.** A highly reflective or busy background behind the hand degrades recognition.
4. **Do not poll it in a hot loop.** It belongs in an event-driven FreeRTOS task that yields, or it
   trips the task watchdog.
5. **Debounce and rate-limit gestures**, or one hand movement fires several actions.

## Course use

| Topic | Role |
|---|---|
| T8 | The new input added *after* the task refactor — proves good structure makes hardware cheap; creates the I2C mutex contention |
| T9 | Local override input in several of the mini-project options |
