# T5 · WiFi Client + HTTP Client — Consuming a Server That Lies to You
**Day 2 · ~2 hours · Participant guide · works in pairs**

## What you'll build
**In plain words:** your board joins someone else's WiFi network and talks to a server on it. It
fetches a setting from that server and uses it, and it sends its own sensor readings back up on a
schedule. The interesting part is what happens when the server misbehaves — because it will.

**The moment it works:** your partner changes a value on their server and, a few seconds later,
your board on the other side of the desk acts on it. Then they unplug their board mid-request and
yours carries on as if nothing happened.

## Objective
By the end you can:
- **Join a network without blocking** — a connect state machine with timeout and backoff, so nothing else on the device stalls waiting for WiFi.
- **Make HTTP requests** — a `GET` that fetches a setting, and a `POST` that uploads your readings.
- **Parse defensively** — status code, then parse error, then field presence and type, before any value is trusted.
- **Fail toward last-good** — a broken server never leaves the device acting on garbage or acting on nothing.
- **Set timeouts that mean something** — a server that accepts your connection and then goes silent must cost you seconds, not forever.

## Knowledge you'll learn first (in this order)
1. **Station mode** — `WiFi.begin()` joins an existing network; SSID, password, and what `WiFi.status()` actually reports.
2. **Why blocking connect is a production bug** — `while (WiFi.status() != WL_CONNECTED)` freezes your sensor, display and alarm.
3. **The connect state machine** — DISCONNECTED → CONNECTING → ONLINE on `millis()`, with a timeout and exponential backoff.
4. **`HTTPClient`** — `begin`, `GET`, `POST`, `getString`, and the `end()` you must call on every path.
5. **Status codes are data** — a negative return is a *transport* failure; `200` vs `4xx` vs `5xx` is an *application* answer. They need different handling.
6. **ArduinoJson** — `JsonDocument`, `deserializeJson`, and the `DeserializationError` you must check.
7. **Defensive field access** — `doc["x"] | fallback`, `is<T>()`, and why a missing key silently reads as `0`.
8. **Timeouts** — `setTimeout()`, and what a silent server does to a device without one.

## Hardware this topic
| Role | Part | Interface | Where |
|---|---|---|---|
| Carrier + WiFi | ROBO ESP32 (onboard radio, station mode) | — | — |
| Environment | Crowtail DHT11 | digital 1-wire | **Grove Port 3 (D26 / D25)** |
| Display | OLED SSD1315 | I²C `0x3C` | Grove Port 2 (D21/D22) |
| Local fallback setting | Rotary Angle | analog | **ADC1 (D32/D33)** |
| Indication | Buzzer + NeoPixel | digital | D23 / D15 |
| Server | your partner's board, or the trainer's reference server | WiFi | the network you join |

> **You work in pairs.** One board runs a server on its own access point; the other joins it and runs
> the client. **Swap roles once the client works** so you both build the client half. If your partner
> is not ready, point your client at the trainer's reference server instead — the client code does
> not care which it is talking to.

## Your requirement
Build a **network client node**, where:
- The board **joins an existing WiFi network** as a station, non-blocking, with backoff on failure.
- It **`GET`s a threshold** from the server on an interval and applies it when the answer is valid.
- It **`POST`s its own readings** — temperature, humidity, uptime — as JSON on an interval.
- The OLED shows link state, the active threshold and **where that threshold came from** (`REMOTE` or `LOCAL`).
- Any server failure — unreachable, slow, `500`, HTML instead of JSON, half a response — means **fall back to the knob**, visibly, and retry later.
- **No network call ever blocks** the sensing, display or alarm path.

> **How you capture this:** create a **new PlatformIO project** (you create it, toolchain proven first —
> `framework/START_PROMPT.md` §0), copy `framework/project_starter.json` into it, and kick off with
> `framework/START_PROMPT.md`. Spend the interview on the endpoint contract and on exactly what the
> device does when each part of that contract is broken.

### Starter interview — suggested answers (T5)
| Area | Your answer |
|---|---|
| Problem | A sensor node that joins a network, takes its configuration from a server, reports its readings back, and keeps working correctly when that server misbehaves. |
| Users *(opt.)* | Deployed sensor reporting to a supervisor system that is not always healthy. |
| Behaviour | Non-blocking join. Every 5–10 s: `POST` readings; every 10–30 s: `GET` the threshold and apply it if valid. OLED shows link state, active threshold and its source. Alarm runs off whichever threshold is currently trusted. |
| Hardware | ROBO ESP32 (onboard WiFi in station mode) + DHT11 + OLED SSD1315 + Rotary Angle; onboard buzzer and NeoPixel. |
| Documents | Attach the DHT11, OLED and Rotary spec cards from `hardware/modules/`, plus the board datasheet — and the server's endpoint URLs and JSON schema. |
| Interfaces | DHT11 = digital 1-wire on **Grove Port 3 (D26 or D25)**; OLED = I²C `0x3C` on D21/D22; knob = **ADC1 (D32/D33) only**. |
| Connectivity | **Station mode** — joins the server's network. `GET /api/config` for the threshold, `POST /api/reading` for telemetry. Plain HTTP, port 80. |
| Constraints | No blocking connect — state machine with timeout and backoff capped ~30 s; explicit `setTimeout()` on every request; `http.end()` on every path including failures; ArduinoJson only, never `String` concatenation; requests on their own `millis()` interval, never in the sensing path. |
| Safety | Sensing and indication only — nothing moves or heats. **Must never happen:** an unvalidated or out-of-range remote threshold being applied, or a slow/unreachable server delaying the sensing and alarm path. Every remote value is range-checked before use; anything invalid is rejected, the local knob value is retained, and the source is shown as `LOCAL`. **Safe state on boot:** local threshold, alarm armed, before any request succeeds. |
| Failure modes | Transport failure, non-200, parse error, missing or wrong-typed field → keep last-good, show `LOCAL`, retry next interval. Silent server → request times out in ≤5 s. Network lost → keep sensing and alarming locally, reconnect with backoff. |
| Reuse *(opt.)* | Any prompt templates you already have that fit — non-blocking timers, sensor reads, display rendering. If you have none yet, this project starts the library. |
| Out of scope | No server code on this board, no MQTT, no TLS, no auth beyond a static token, no OTA. |
| Acceptance | Joins the network without ever blocking the loop; applies a valid remote threshold and shows `REMOTE`; survives `500`, HTML body, truncated JSON and a silent server with no crash and no bad action; falls back to `LOCAL` visibly; POSTs well-formed JSON the server accepts; alarm latency unaffected throughout. |

## Flow (stages)
- **Stage 0 — Find your server (10 min):** agree who is serving. Note the SSID, password, the board's IP, the two endpoint paths and the JSON schema. Confirm from a laptop with `curl` that the server actually answers **before** writing any client code.
- **Stage 1 — Join without blocking (20 min):** prompt for a station connect as a state machine with timeout and backoff. **Test:** the OLED keeps refreshing and the alarm still fires *while* it is connecting — and when you give it a wrong password, nothing on the device freezes. Save a **wifi-station-nonblocking** template.
- **Stage 2 — First GET (20 min):** fetch the config JSON and print the threshold to serial. **Test:** partner changes the value on their server, you see the new number arrive. Save an **http-client-get** template.
- **Stage 3 — Break the server on purpose (30 min):** ask your partner to make their endpoint return `500`, then an HTML page, then half a JSON object, then nothing at all. Harden the parse: transport → status → `DeserializationError` → field presence and type → last-good fallback. **Test:** all four hostile modes handled, alarm never disturbed. **This stage is the topic.**
- **Stage 4 — POST your readings (25 min):** serialize temp, humidity and uptime with ArduinoJson and POST on a non-blocking interval. **Test:** your partner sees your records arriving on their server, and the values track when you breathe on your sensor.
- **Stage 5 — Pull the plug (15 min):** partner powers down their board mid-request. **Test:** your request times out within 5 s, the display flips to `LOCAL`, the alarm keeps working on the knob value, and it recovers by itself when their board comes back.

## Catch the AI
- ⚠ **Blocking connect.** It will write `while (WiFi.status() != WL_CONNECTED) { delay(500); }`. If the network is absent that is forever — your alarm goes deaf waiting. Demand a state machine.
- ⚠ **The fragile parse — the headline trap.** It writes `doc["threshold"]` with no status check and no `DeserializationError` check. Ask *"what if the server returns 500, or an HTML login page?"* then actually make that happen. Its code reads `0` and your alarm fires at 0 °C forever.
- ⚠ **No timeout.** The default behaviour against a server that accepts the connection then says nothing is a long stall. Demand an explicit `setTimeout()`.
- ⚠ **Treating `-1` as an HTTP status.** A negative return is a transport failure, not a response code — they need different handling.
- ⚠ **`String` concatenation to build JSON** — heap fragmentation on a device meant to run for weeks. Demand ArduinoJson serialization.
- ⚠ **Missing `http.end()`**, or a fresh `begin()` every loop — connections leak until the device stops working.
- ⚠ **Assuming the server is up.** Its happy-path code has no branch for "no reply at all", which is the most common real condition.
- ⚠ **Hardcoded SSID and password** in the source, often printed to serial at boot.

## Done when (shared objective)
- [ ] Joins the network with **no blocking loop** — sensor, display and alarm run throughout, including on a wrong password.
- [ ] **`GET` applies a valid remote threshold**; OLED shows `REMOTE` vs `LOCAL`.
- [ ] **`POST` sends well-formed JSON** your partner's server accepts, with live values.
- [ ] Survives **`500`, HTML instead of JSON, truncated JSON and a silent server** — no crash, no hang, no acting on garbage.
- [ ] Falls back to the knob **visibly** and recovers by itself when the server returns.
- [ ] Every request has an explicit timeout; `http.end()` on all paths; alarm never blocked.

## Save to your prompt library
- `wifi-station-nonblocking` template · `http-client-get` template · `http-client-post` template · `defensive-json-parse` template — the last one is the most reusable thing you will write all week.
