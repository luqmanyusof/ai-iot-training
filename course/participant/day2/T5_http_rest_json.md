# T5 · HTTP/REST & JSON — Talking to a Server That Lies to You
**Day 2 · ~2 hours · Participant guide**

## What you'll build
**In plain words:** your monitor starts talking to a web server. It sends its readings up on a
schedule, and asks the server what comfort limit to use. The interesting part is what happens when
the server misbehaves — an error, a web page instead of data, half an answer, or no reply at all.
The device notices, ignores the bad answer, falls back to the knob, and says so on screen.

**The moment it works:** the server is broken on purpose in four different ways, and the device
keeps behaving correctly through every one of them.

## Objective
Talk to a server that lies to you. By the end you can:
- **Speak HTTP deliberately** — verbs, status codes, and the difference between a transport failure and an application answer.
- **Parse JSON defensively** — status check → `DeserializationError` → field presence/type → fallback; never act on garbage.
- **Set explicit timeouts** — a silent server must cost you 5 seconds, not forever.
- **Fail toward last-good** — server down or nonsense returned → the device keeps working on what it last knew.
- **Prove it live** — your device survives a 500, an HTML page, truncated JSON and a hang without crashing or acting on bad data.

## Knowledge you'll learn first (in this order)
1. **HTTP in one minute** — verbs, status codes, headers, request/response as a *synchronous* model.
2. **`HTTPClient` on ESP32** — `begin`/`GET`/`POST`/`getString`/`end`, and why `end()` matters.
3. **Status codes are data** — a negative return is a *transport* failure; 200 vs 4xx vs 5xx is an *application* answer. They need different handling.
4. **JSON with ArduinoJson** — `JsonDocument`, `deserializeJson`, and the `DeserializationError` you must check.
5. **Defensive field access** — `doc["x"] | fallback`, `is<T>()`, and why a missing key silently becomes `0`.
6. **Timeouts** — `setTimeout()`; a server that accepts your TCP connection and then says nothing forever.
7. **Payload budget** — why you size the document and don't build JSON with `String` concatenation.

## Hardware this topic
| Role | Part | Where |
|---|---|---|
| Carrier + WiFi | ROBO ESP32 | — |
| Display | OLED SSD1315 | I²C `0x3C`, Grove Port 2 (D21/D22) |
| Environment | Crowtail DHT11 | digital 1-wire |
| Local threshold | Rotary Angle | ADC1 (D32/D33) |
| Alarm | Buzzer + NeoPixel | D23 / D15 |
| Endpoint | test REST endpoint (trainer-provided) | network |

## Your requirement
Extend your monitor so that:
- It **POSTs a JSON telemetry record** — temp, humidity, threshold, state, uptime — on a fixed non-blocking interval.
- It **GETs a remote threshold** that overrides the knob when the server provides a valid one.
- The OLED shows **which source is active** — `LOCAL` or `REMOTE`.
- Any server failure — unreachable, slow, 500, HTML, truncated JSON — means **fall back to last-good / knob**, visibly.
- **No network call ever blocks the alarm.**

> **How you capture this:** new PlatformIO project (you create it, toolchain proven first — `framework/START_PROMPT.md` §0), starter in, kick off with `framework/START_PROMPT.md`. "Same as T4" covers the hardware; the
> new answers are the endpoint contract and the hostile-response behaviour.

### Starter interview — suggested answers (T5)
| Area | Your answer |
|---|---|
| Problem | The monitor exchanges data with a flaky REST API — readings up, threshold down — without ever acting on garbage. |
| Users *(opt.)* | Facilities IT REST API; returns 500s during deploys and an HTML SSO page when sessions expire. |
| Behaviour | POST telemetry every 5–10 s; GET config; a valid `threshold` overrides the knob; anything invalid → last-good + `LOCAL` badge. |
| Hardware | Same as T4. |
| Documents | Same as T4 + the trainer's endpoint URL and JSON schema. |
| Interfaces | POST `{"temp":..,"humidity":..,"threshold":..,"state":"..","uptime_s":..}`; GET expects `{"threshold":N}`. |
| Connectivity | WiFi + HTTP to the trainer's test endpoint. |
| Constraints | ArduinoJson only (no `String` concatenation); gate every response transport→status→parse→type; `setTimeout(5000)`; `http.end()` on all paths; non-blocking interval. |
| Safety | **Low risk, but the blast radius grew:** a remote value now influences behaviour. A malformed or out-of-range threshold must never silently disable the alarm — reject it, fall back to the knob, and show `LOCAL` so the operator knows. |
| Failure modes | 500 / HTML / truncated / silent → no crash, no bad action; last-good + `LOCAL`; retry later; alarm unaffected. |
| Reuse *(opt.)* | T4 REQUIREMENTS; wifi-nonblocking + secrets-in-nvs templates. |
| Out of scope | No MQTT (T6), no TLS, no auth flows beyond a static token. |
| Acceptance | Valid POSTs on interval; remote threshold applied + shown; survives all four hostile modes; falls back visibly; alarm never blocked. |

## Flow (stages)
- **Stage 0 — Endpoint up (10 min):** point your board at the trainer's test endpoint; confirm you can reach it. Note the exact URL and expected schema.
- **Stage 1 — First GET (25 min):** prompt the AI for an HTTP GET that fetches a JSON config and prints it. It will probably work first time — on the happy path. Save an **http-get-json** template.
- **Stage 2 — Break the happy path (30 min):** ask the trainer to make the endpoint return **500**, then an **HTML error page**, then **half a JSON object**. Watch what your code does. Now prompt for hardened parsing: check the status code, check `DeserializationError`, validate the field exists and has the right type, and keep the last-good value on failure. This stage is the topic.
- **Stage 3 — POST telemetry (25 min):** prompt to serialize your live readings into JSON and POST them on a **non-blocking interval** (~5–10 s). Verify the server sees well-formed records.
- **Stage 4 — Remote override (20 min):** apply the fetched threshold over the knob value, with a clear rule for which wins and an OLED indicator showing the source (`LOCAL` / `REMOTE`).
- **Stage 5 — Failure injection (10 min):** pull the network mid-POST. The device must time out, keep alarming on local data, revert to `LOCAL` threshold, and retry later.

## Catch the AI
- ⚠ **The fragile parse.** The AI will write `doc["threshold"]` with no status-code check and no `DeserializationError` check. Ask it: *"what if the server returns 500, or an HTML login page?"* — then actually make that happen. Its code will read `0` and your alarm will fire at 0 °C forever.
- ⚠ **No timeout.** Default behaviour on a silent server is a long stall. Demand an explicit `setTimeout()`.
- ⚠ **`String` concatenation to build JSON** — heap fragmentation on a long-running device. Demand ArduinoJson serialization.
- ⚠ It often **leaks the connection** by skipping `http.end()`, or re-`begin()`s per loop.

## Done when (shared objective)
- [ ] POSTs valid JSON telemetry on a non-blocking interval.
- [ ] GETs and applies a remote threshold; OLED shows `LOCAL` vs `REMOTE`.
- [ ] Survives **500**, **HTML instead of JSON**, **truncated JSON** and **timeout** without crashing, hanging, or acting on garbage.
- [ ] Falls back to the last-good / knob value and says so on the display.
- [ ] Alarm never blocked by a network call.

## Save to your prompt library
- `http-get-json` template · `defensive-json-parse` template · `http-post-telemetry` template.
