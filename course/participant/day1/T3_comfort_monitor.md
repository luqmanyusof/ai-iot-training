# T3 · I²C Display + Digital Sensor → Smart Comfort Monitor
**Day 1 · ~2 hours · Participant guide**

## What you'll build
**In plain words:** a room comfort monitor. A small screen shows the current temperature and
humidity. You turn a knob to set a comfort limit, which also shows on the screen. When the room
crosses that limit, the light turns red and a buzzer sounds.

**The moment it works:** turn the knob down past the current room temperature — the alarm fires
straight away, and the screen carries on updating the whole time instead of freezing.

## Objective
Three parts, one non-blocking device. By the end you can:
- **Drive an I²C bus** — scan it, find your device's address, and prove the wiring before writing driver code.
- **Read a 1-wire digital sensor** — DHT11 is a slow (~1 Hz) digital-pin protocol, *not* I²C; catching the AI on this is part of the topic.
- **Run three jobs on independent `millis()` timers** — sensor, display, logic — with nothing freezing anything else.
- **Compose several parts into one device** — a filtered analog read, a slow sensor and a display, all sharing one loop without fighting.
- **Prove it locally first** — a device that is fully trustworthy on the bench before any network is anywhere near it.

## Knowledge you'll learn first (in this order)
1. **I²C in one minute** — SDA/SCL, device addresses, why many devices share two wires; run an **I²C scanner**.
2. **The DHT 1-wire protocol** — DHT11 is a single digital pin (not I²C), and it's **slow** (~1 Hz).
3. **Non-blocking multitasking** — running sensor + display + logic on independent `millis()` timers.
4. **A small state machine** — the OK / WARN / ALARM pattern, and why thresholds need hysteresis.

## Hardware this topic
| Role | Part | Interface | Where |
|---|---|---|---|
| Environment | Crowtail DHT11 | digital 1-wire | **Grove Port 3 (D26 / D25)** |
| Display | OLED SSD1315 | I²C `0x3C` | Grove Port 2 (D21/D22) |
| Threshold knob | Rotary Angle | analog | ADC1 (D32/D33) |
| Alarm | Buzzer + NeoPixel | digital | D23 / D15 |

## Your requirement
Build a Smart Comfort Monitor where:
- The OLED shows **live Temp and Humidity**, refreshing every ~2 s.
- The knob sets a **comfort threshold**, also visible on the OLED.
- Crossing the threshold drives **OK / WARN / ALARM** — NeoPixel colour + buzzer.
- Everything is **non-blocking** — the display never freezes while the sensor reads.
- A failed sensor read is handled — keep the last good value and flag it, never crash.

> **How you capture this:** new PlatformIO project (you create it, toolchain proven first — `framework/START_PROMPT.md` §0), starter in, kick off with `framework/START_PROMPT.md`, run the interview — your hardware
> answers now span three datasheets. Approve `REQUIREMENTS.md` + `PHASES.md` before coding.

### Starter interview — suggested answers (T3)
| Area | Your answer |
|---|---|
| Problem | A local comfort monitor: live temp/humidity on an OLED with a knob-set threshold and an audible/visual alarm. |
| Users *(opt.)* | Room occupants / facilities; indoor room. |
| Behaviour | Read DHT11 every 2 s → render OLED; knob sets threshold; reading vs threshold → OK/WARN/ALARM → NeoPixel + buzzer. |
| Hardware | ROBO ESP32 + DHT11 + OLED SSD1315 + Rotary Angle; onboard buzzer + NeoPixel. |
| Documents | Attach `hardware/modules/Grove-TemperatureAndHumidity_Sensor-DHT11.md`, `hardware/modules/Grove-OLED-Display-0.96-SSD1315.md`, `hardware/modules/Grove-Rotary_Angle_Sensor.md` and the board datasheet. |
| Interfaces | DHT11 = **digital 1-wire on Grove Port 3 (D26 or D25)**; OLED = I²C `0x3C` on D21/D22 (SSD1306-compatible driver); rotary = ADC1. |
| Connectivity | None — deliberately local. Everything must be trustworthy on the bench first. |
| Constraints | DHT11 is NOT I²C (`0x38` is the DHT20 — a different part); poll no faster than 1–2 s; independent `millis()` timers; no `delay()` anywhere. |
| Safety | Sensing and indication only — nothing moves or heats. **Must never happen:** the alarm firing on an invalid or missing sensor reading, or a failed read being treated as `0`. **Safe state on boot:** quiet, OK state, alarm suppressed until the first valid reading arrives. A failed or NaN read holds the last good value and is flagged as stale rather than acted on. |
| Failure modes | NaN/failed DHT read → keep last good + flag it; display keeps refreshing regardless. |
| Reuse *(opt.)* | Any prompt templates you already have that fit — an analog-read or moving-average template, or a state-machine pattern. If you have none yet, this project starts the library. |
| Out of scope | No network, no cloud, no logging. |
| Acceptance | Live values update; threshold visible and knob-driven; alarm fires within one refresh of crossing; zero `delay()`; display never freezes. |

## Flow (stages)
- **Stage 0 — I²C scan (10 min):** prompt for an I²C scanner; confirm **`0x3C`** (OLED) appears.
- **Stage 1 — OLED hello (20 min):** prompt to init the SSD1315 (SSD1306-compatible) and print two text lines.
- **Stage 2 — DHT11 read (20 min):** prompt to read DHT11 every 2 s on its digital pin (**Grove Port 3, D26 or D25** — tell the AI which one you wired); print to serial.
- **Stage 3 — Render (20 min):** show `Temp: xx.x C` and `Humidity: xx %` on the OLED, refreshed non-blocking.
- **Stage 4 — Threshold (20 min):** add the Rotary knob to set a threshold; filter it and display it.
- **Stage 5 — Comfort logic (25 min):** compare reading to threshold → OK/WARN/ALARM → NeoPixel colour + buzzer, all in one non-blocking loop.

## Catch the AI
- ⚠ **DHT11 is not I²C** — if the AI tries to read it over I²C or at address `0x38` (that's DHT20), it's wrong. It's a digital pin with the DHT library.
- ⚠ The AI may use `delay(2000)` for the DHT read — that freezes the OLED. Force `millis()` timing.

## Done when (shared objective)
- [ ] OLED shows live Temp + Humidity, updating.
- [ ] Knob visibly sets the threshold on the OLED.
- [ ] Crossing the threshold triggers buzzer + red NeoPixel within one refresh.
- [ ] No `delay()` anywhere; display never freezes.

## Save to your prompt library
- `non-blocking-timer` template · `i2c-oled-display` template.
