---
title: Analog Micro Servo 9g (TS90A, 3V–6V)
description: TS90A 9g analog micro servo — spec card for the ESP32 AI-IoT course
part_type: actuator
interface: PWM
model: T-TS90A
vendor: Cytron
product_url: https://my.cytron.io/p-analog-micro-servo-9g-3v-6v
price_myr: 12.00
price_myr_bulk_50: 9.60
card_status: COURSE SPEC CARD — compiled from the vendor product page, not a manufacturer datasheet
---

# Analog Micro Servo 9g — TS90A (3 V–6 V)

> **This is a course-written spec card, not a vendor datasheet.** The TS90A is a generic hobby part
> with no manufacturer wiki page. Everything here comes from the Cytron product listing. Writing a
> card like this *is* the datasheet-prep skill the framework teaches — the AI is not allowed to
> assume anything that isn't on this page.

![TS90A analog micro servo](https://static.cytron.io/image/cache/catalog/products/TS90A/TS90A%20(5)-800x800.jpg)

## What it is
A 9 g analog hobby servo with a standard **3-wire servo lead**. **It is not a Grove module** — it
plugs directly onto one of the ROBO ESP32's 3-pin servo headers (D4 / D5 / D18 / D19).

**Chosen specifically for this kit because it runs from 3.0 V** — the ESP32 is a 3.3 V part, and the
common SG90 (4.8–5 V) is marginal here. This is the servo Cytron itself recommends for 3.3 V
microcontrollers.

## Specification (vendor-stated)

| Parameter | Value |
|---|---|
| Model | **T-TS90A** |
| **Operating voltage** | **3.0 – 6.0 V DC** |
| Control system | Analog |
| **PWM pulse width** | **500 µs – 2500 µs** |
| Rotation range | **180°** |
| Stall torque | 1.3 kg·cm (18.09 oz·in) @ 4.8 V |
| Speed | 0.12 s / 60° @ 4.8 V |
| Weight | 9 g |
| Gears | Plastic |
| Motor | Metal |
| Dimensions | 23.2 × 12.5 × 22 mm |
| Lead length | 20 cm |

## Wiring — check before powering up

| Wire colour | Function | ROBO ESP32 servo header |
|---|---|---|
| **Brown** | GND | GND pin |
| **Red** | +V (3–6 V) | V+ pin |
| **Orange** | Signal (PWM) | Signal pin |

⚠ **The connector is not keyed against reversal.** Confirm the brown wire lands on GND before
applying power — a reversed plug is the most common first-time servo mistake.

## Control

- Library: **`ESP32Servo`** — the stock Arduino `Servo.h` **will not compile** on ESP32.
- Pulse range is **500–2500 µs** for 0–180°. If you set custom limits, use:
  `servo.attach(pin, 500, 2500);`
- Sweep slowly the first time and find your unit's real end-stops — individual units vary, and
  driving past a mechanical limit stalls the motor.

## Power on the ROBO ESP32

Servo headers take **V+ = Vservo = the board's supply rail**, not a regulated 5 V:

| Power source | Vservo |
|---|---|
| USB only | VUSB − 0.4 ≈ **4.6 V** |
| LiPo or VIN only | VLiPo or VIN |

Both sit comfortably inside the 3–6 V range, so **no external supply or level shifting is needed** —
unlike the 4.8 V-minimum SG90.

⚠ **Do not power a servo from the 3V3 Grove rail.** It is limited to 300 mA total across all Grove
ports; use the servo header.

## Documented gotchas

1. **Stock `Servo.h` does not work on ESP32** — use `ESP32Servo`. The AI recommends the wrong one
   with high confidence, every time.
2. **Blocking sweeps.** A `for` loop stepping degrees with `delay(15)` stalls everything else for
   roughly a second. Servo motion must be non-blocking.
3. **Inrush current.** Start-up current can brown-out the board on USB power and reset the ESP32
   mid-move. Unexplained reboots when the servo starts = suspect supply first, not code.
4. **No position feedback.** The servo tells you nothing about where it actually is. If the horn is
   blocked, your code still believes the commanded angle — any fail-safe must exist in software.
5. **Modest torque** (1.3 kg·cm). Fine for a demo vent flap or pointer; it will stall against real
   mechanical load, and a stalled servo draws heavy current continuously.

## Course use

| Topic | Role |
|---|---|
| **T7 · UART & RS485** | Driven by the **partner board's** rotary knob across the bus; fail-safe position on link loss |
| **T9 · Capstone** | Vent/damper actuator driven by comfort state and validated cloud commands, inside a FreeRTOS task |

## Pre-delivery checklist (trainer)

- [ ] Confirm smooth travel across the full 500–2500 µs range on one unit.
- [ ] Confirm no board brown-out when the servo starts under USB power.
- [ ] Note the real end-stops for the batch you bought.
