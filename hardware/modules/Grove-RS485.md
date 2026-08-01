---
title: Grove - RS485
description: Half-duplex differential serial transceiver on UART — course spec card
part_type: comms
interface: UART (differential, half-duplex)
source: https://wiki.seeedstudio.com/Grove-RS485/
card_status: COURSE SPEC CARD — written for this course. Not vendor content. Verify anything marked VERIFY on real hardware.
---

# Grove - RS485

> **Course-written spec card.** The vendor's full documentation is at the link above — this page
> carries only what this course needs, plus the ESP32-specific facts the vendor page does not cover.
> Attach this when the starter interview asks for a datasheet.

## What it is
An RS485 transceiver on a Grove connector. It carries **ordinary UART bytes over a differential
pair**, giving noise immunity and long cable runs. It is not a protocol — framing is yours to write.

## Specification

| Parameter | Value |
|---|---|
| Interface | **UART** (to the host), RS485 differential A/B (to the bus) |
| Supply voltage | **3.3 V / 5 V — ESP32-safe** |
| Error-free rate | 500 kbps |
| Reach | up to 1200 m at lower data rates |
| Duplex | **Half-duplex** — one talker at a time |

## On the ROBO ESP32

- **UART Grove port (D16/D17)** — `Serial2`.
- `Serial2.begin(baud, SERIAL_8N1, RX, TX)` — **pin order matters**; reversed gives silence.
- Bus wiring between nodes: **A to A, B to B** on a twisted pair.
- No library. Plain `HardwareSerial`.

## VERIFY on your first unit

The vendor page does not state these in text (the pinout is an image). Confirm before class:

- [ ] Is there a **DE/RE direction pin**, or is the transceiver auto-direction?
- [ ] Is a **120 ohm termination resistor** fitted, a solder pad, or absent?

## Gotchas

1. **Half-duplex is the whole lesson.** Two nodes driving one pair at once produces garbage. The
   pair must agree who may talk when — master-polls or strict time slots.
2. **`flush()` before switching to receive**, or you drop the reply and blame the wiring.
3. **Nothing frames your data.** A stream gives you bytes, not messages: delimiter, address, length,
   payload, checksum are all yours.
4. **A/B polarity swapped = silence, not an error.** Same for a baud mismatch.
5. **The AI treats it as plain `Serial`** and, when frames garble, suggests "add a delay". That is
   not a fix.

## Course use

| Topic | Role |
|---|---|
| T7 | Cross-control: your knob drives your partner's servo over the bus |
| T9 | Optional two-node gateway project |
