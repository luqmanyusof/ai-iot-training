---
title: Grove - OLED Display 0.96" (SSD1315)
description: 128x64 monochrome I2C OLED — course spec card
part_type: display
interface: I2C
source: https://wiki.seeedstudio.com/Grove-OLED-Display-0.96-SSD1315/
card_status: COURSE SPEC CARD — written for this course. Not vendor content. Verify anything marked VERIFY on real hardware.
---

# Grove - OLED Display 0.96" (SSD1315)

> **Course-written spec card.** The vendor's full documentation is at the link above — this page
> carries only what this course needs, plus the ESP32-specific facts the vendor page does not cover.
> Attach this when the starter interview asks for a datasheet.

## What it is
A 128x64 monochrome (white) passive-matrix OLED on a Grove I2C connector.

## Specification

| Parameter | Value |
|---|---|
| Interface | **I2C** |
| **I2C address** | **`0x3C`** (changeable on the module) |
| Resolution | 128 x 64 pixels, monochrome white |
| Input voltage | **3.3 V / 5 V** — onboard level shifting, ESP32-safe |
| Controller | **SSD1315** |
| Operating temperature | -40 C to +85 C |

## On the ROBO ESP32

- **Grove Port 2 (D21/D22)** — the board's only I2C Grove port.
- Used **T3-T7**. From **T8** the gesture sensor takes this port and the device goes headless.

## Library

`U8g2` **or** `Adafruit_SSD1306` + `Adafruit_GFX`.

```cpp
// U8g2, hardware I2C
U8G2_SSD1306_128X64_NONAME_F_HW_I2C u8g2(U8G2_R0, U8X8_PIN_NONE);
```

## Gotchas

1. **SSD1315 is driven by SSD1306 drivers.** There is no separate "SSD1315" Arduino library and the
   AI may insist you need one. Use the SSD1306 constructor.
2. **Address is `0x3C`, not `0x3D`.** Scan the bus and confirm before writing driver code.
3. **Full-buffer modes cost RAM** (`_F_` in U8g2). On a FreeRTOS task that stack/heap cost is real.
4. **Rendering is slow relative to a sensor loop** — refresh on its own timer, never in the hot path.

## Course use

| Topic | Role |
|---|---|
| T3 | Live temp/humidity + threshold display |
| T4-T6 | Link state, RSSI, threshold source |
| T7 | Remote knob value, RSSI, STALE flag |
