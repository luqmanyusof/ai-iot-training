# T4 · WiFi — Connecting Without Blocking
**Day 2 · ~2 hours · Participant guide**

## What you'll build
**In plain words:** a room monitor that is also on WiFi. A small screen shows temperature, humidity,
a knob-set comfort limit, and the network status. The real point is what happens when the network
isn't there: the device keeps measuring, displaying and alarming regardless, and rejoins by itself
when WiFi returns.

**The moment it works:** switch the hotspot off mid-run. Nothing on the device stutters. Switch it
back on and it reconnects without you touching it.

## Objective
Put a working local monitor online **without breaking any of it**. By the end you can:
- **Reject the blocking connect** — the `while(!connected) delay()` the AI always writes, and why it's a production bug.
- **Build a connection state machine** — DISCONNECTED → CONNECTING → ONLINE on `millis()`, with timeout and exponential backoff.
- **Keep secrets out of source** — SSID/PSK provisioned into NVS; nothing readable in `main.cpp`.
- **Survive an AP outage** — the device degrades to local-only and rejoins unaided, no reboot.
- **See the analog trap fire** — an analog input on the wrong pin reads garbage the instant the radio starts, and it looks exactly like a software bug.

## Knowledge you'll learn first (in this order)
1. **Station mode basics** — SSID/PSK, DHCP, what `WiFi.status()` actually returns.
2. **Why blocking connect is a production bug** — `while(!connected)` freezes your sensor, display and alarm.
3. **The connection state machine** — DISCONNECTED → CONNECTING → ONLINE, driven by `millis()`.
4. **Reconnect with backoff** — why hammering the AP every loop is worse than waiting.
5. **WiFi events** — `WiFi.onEvent()` as the callback alternative to polling.
6. **Secrets out of source** — credentials in NVS (`Preferences`) or a git-ignored header, never hardcoded in `main.cpp`.
7. **The ADC2 conflict** — the radio takes ownership of ADC2, so any analog input there stops working once WiFi is up.

## Hardware this topic
| Role | Part | Where |
|---|---|---|
| Carrier | ROBO ESP32 (onboard WiFi) | — |
| Status | OLED SSD1315 | I²C `0x3C`, Grove Port 2 (D21/D22) |
| Environment | Crowtail DHT11 | digital 1-wire |
| Threshold knob | Rotary Angle | **ADC1 (D32/D33)** |
| Alarm | Buzzer + NeoPixel | D23 / D15 |

## Your requirement
Build a **network-aware room monitor**, where:
- WiFi connects **in the background** — sensor, display and alarm run the entire time.
- The OLED shows **connection state and RSSI**.
- Pulling the AP away changes nothing locally — and the device **rejoins by itself** when it returns.
- Reconnect uses **backoff**, not per-loop hammering.
- **No credentials are readable in the source file.**

> **How you capture this:** new PlatformIO project (you create it, toolchain proven first —
> `framework/START_PROMPT.md` §0), starter in, kick off with `framework/START_PROMPT.md`. Spend the
> interview on connectivity, secrets and failure behaviour — that is where this build is decided.

### Starter interview — suggested answers (T4)
| Area | Your answer |
|---|---|
| Problem | A room monitor that joins WiFi in the background and never stops sensing, displaying or alarming when the network misbehaves. |
| Users *(opt.)* | Wall-mounted facility monitor; the AP is not trustworthy. |
| Behaviour | Read DHT11 every 2 s → OLED shows temp, humidity, knob threshold, link state and RSSI; alarm on threshold crossing. Connect runs as a background state machine; local behaviour is identical online or offline. |
| Hardware | ROBO ESP32 (onboard WiFi) + DHT11 + OLED SSD1315 + Rotary Angle; onboard buzzer and NeoPixel. |
| Documents | Attach the DHT11, OLED and Rotary spec cards from `hardware/modules/`, plus the board datasheet. |
| Interfaces | DHT11 = digital 1-wire; OLED = I²C `0x3C` on D21/D22; knob = **ADC1 (D32/D33) only** — it must still read correctly with the radio on. |
| Connectivity | WiFi station mode only — no HTTP, no MQTT, no cloud. |
| Constraints | No blocking connect; backoff 1→2→4 s… capped ~30 s; creds in NVS via `Preferences`; sensor/display timers independent of link state. |
| Safety | Sensing and indication only, but it runs unattended for long periods. **Must never happen:** alarm behaviour being altered, suppressed or delayed as a side effect of network state. **Safe state on boot:** quiet, OK, local threshold, alarm armed — independent of link status. Connection handling must not block or gate the sensing and alarm path. |
| Failure modes | AP gone → keep monitoring + alarming locally, no reboot; rejoin unaided; alarm latency unaffected. |
| Reuse *(opt.)* | Any prompt templates you already have that fit — a non-blocking-timer template, or an existing sensor/display spec. If you have none yet, this project starts the library. |
| Out of scope | No HTTP, MQTT, cloud, captive portal, or OTA. |
| Acceptance | No blocking loop anywhere; display/alarm live during connect; auto-rejoin after drop; creds in NVS; knob stable with WiFi on. |

## Flow (stages)
- **Stage 0 — Local baseline (10 min):** get sensor, display, knob and alarm working with **no radio code at all**. Confirm it is solid *before* WiFi enters the picture — this is your known-good reference, and you will need it.
- **Stage 1 — Naive connect (25 min):** prompt the AI for "connect the ESP32 to WiFi and print the IP." Take whatever it gives you. Flash it into your monitor and **watch the OLED freeze** while it connects. Now ask it: *"what breaks in production?"*
- **Stage 2 — Non-blocking state machine (30 min):** prompt to rewrite the connect as a state machine with a connect timeout and **exponential backoff** on retry. The DHT read and OLED refresh must continue throughout. Save this as your **wifi-nonblocking** prompt template.
- **Stage 3 — Secrets out of source (20 min):** prompt to move SSID/password into NVS via `Preferences`, with a serial command to provision them once. Nothing secret left in `main.cpp`.
- **Stage 4 — Break it on purpose (20 min):** show state + RSSI on the OLED. Then **turn off the hotspot**. The device must detect the drop, keep alarming on local data, and rejoin by itself when the AP returns — no reboot, no freeze.
- **Stage 5 — The ADC2 reckoning (15 min):** with WiFi **on**, turn your knob. If the threshold still tracks smoothly, the knob is on ADC1. If it reads garbage or zero, it is on ADC2 — move it and log the catch.

## Catch the AI
- ⚠ The AI will almost certainly emit **`while (WiFi.status() != WL_CONNECTED) { delay(500); }`**. This blocks forever if the AP is absent — your alarm goes deaf while it waits. Reject it and demand a state machine.
- ⚠ It will **hardcode your SSID and password** in the source. Call it out: that is a credential leak the moment the repo is shared.
- ⚠ It usually has **no reconnect path at all** — it connects once in `setup()` and assumes the network is eternal. Ask what happens at hour 300.

## Done when (shared objective)
- [ ] Device connects to WiFi with **no blocking loop** — sensor, display and alarm run throughout.
- [ ] OLED shows connection state and RSSI.
- [ ] AP pulled away → device keeps working locally and **rejoins unaided**, no reboot.
- [ ] Credentials are in NVS, not in source.
- [ ] Knob still reads cleanly **with the radio on** (ADC1 proven).

## Save to your prompt library
- `wifi-nonblocking` template · `secrets-in-nvs` template.
