# Outline ⇄ Kit Lesson Mapping (revised)

How the final kit maps onto the 3-day outline, and what changed vs `esp32_session.json`.
Audience re-leveled for **experienced engineers**: less setup hand-holding, more protocol/architecture depth.

## Interface-lesson coverage
| Lesson / skill | Hardware that teaches it |
|---|---|
| Digital I/O + interrupt + debounce | **Onboard buttons D34/D35** (input-only trap) + buzzer D23 |
| Analog + ADC1-vs-WiFi trap + filtering | **Rotary Angle** on ADC1 (D32/D33) |
| I2C bus + addressing | **OLED + Gesture** (2 devices on the two native I2C connectors) |
| Digital 1-wire sensor protocol | **Crowtail DHT11** (digital Grove port, DHT library) |
| Industrial serial bus + UART | **Grove RS485** (differential, half-duplex, own framing + addressing) |
| Event/gesture input (touchless UI) | **Gesture PAJ7660** (I2C, 15+ gestures) |
| PWM actuation | **TS90A micro servo** (3–6 V, ESP32Servo; 3-pin lead onto the servo header) + onboard motor driver |
| Remote actuation (MQTT two-way) | **Onboard buzzer / NeoPixel / Servo** (relay dropped) |

## Changes vs original JSON plan
| # | Original (JSON) | Now | Why |
|---|---|---|---|
| 1 | DHT11/DHT20 | **Crowtail DHT11 (digital)** | Grove-compatible; teaches the digital 1-wire protocol (I2C addressing now shown by OLED + Gesture) |
| 2 | Grove Light/LDR (analog) | **Rotary Angle (analog)** | Cheaper analog to keep ADC1 lesson after LDR dropped |
| 3 | Ultrasonic ranging | **Grove RS485 (comms)** | Ultrasonic and ToF both dropped (budget); no distance sensor in the kit. RS485 restores the UART lesson and adds a differential industrial bus. Replaced the LoRa Radio (budget) — and the Grove CAN GD32E103 was rejected as 5 V-only, incompatible with the 3.3 V ROBO Grove ports. |
| 4 | Relay actuator | **Onboard buzzer/NeoPixel/servo** | ROBO already has ample outputs; saves cost + a port |
| 5 | PIR / presence | **Removed** | Capstone pivots to ranging/touchless control, not occupancy |
| 6 | (none) | **Gesture PAJ7660 (I2C)** | Indoor-safe touchless-control lesson; replaces the GPS (GPS can't get a fix indoors) |
| 7 | OTA + rollback (Day 3) | **Removed** | Dropped by decision; frees Day-3 time for RTOS + gesture + capstone. **Outcome (b) no longer claims OTA.** |

## Per-module → outline module
| Kit module | Primary outline module(s) |
|---|---|
| Onboard buttons/buzzer/NeoPixel | 1.1 blink, 1.3 GPIO/interrupt/state machine, 2.1 WiFi status LED |
| Rotary Angle | 1.4 Analog sensors & ADC (filtering, calibration) |
| DHT11 + OLED | 1.5 Smart Room Monitor; 3.1 FreeRTOS task split |
| Grove RS485 | Day-3 cross-control over UART: half-duplex contention, addressing, framing, checksum, actuator fail-safe |
| Gesture PAJ7660 | Day-3 touchless control; capstone interactive input |
| TS90A Micro Servo | T7 cross-control actuation; T9 capstone vent/damper with fail-safe |

## Interrupt lesson note
Doppler and GPS were dropped, and the relay was dropped, so the **hardware-interrupt lesson lives on the onboard buttons (D34/D35)** — which also doubles as the "AI wrongly writes `INPUT_PULLUP`" teaching moment. The **UART lesson is back** via the Grove RS485 module (D16/D17) after GPS was removed.

## Standing constraints to bake into every prompt
- Analog → **ADC1 only** (WiFi-safe).
- Buttons D34/D35 → **input-only, no pull-up**.
- TS90A servo → **3.0–6.0 V, ESP32-safe**; **ESP32Servo**, not stock `Servo.h`; **not a Grove part** — 3-pin lead onto the servo header, brown=GND/red=V+/orange=signal; PWM 500–2500 µs.
- 2 I2C devices → **no hub needed**; OLED (0x3C) on Grove Port 2, Gesture (0x73, verify) on Maker/QWIIC. DHT11 is digital, not on the bus.
- RS485 → **3.3 V/5 V, ESP32-safe**; UART on D16/D17; 2 nodes per bus, A→A and B→B; **half-duplex — agree who may talk when**; matched baud both ends; 120 Ω termination at each end of the trunk.
