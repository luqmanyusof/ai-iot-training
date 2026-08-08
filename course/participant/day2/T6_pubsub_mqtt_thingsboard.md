# T6 · Pub/Sub → MQTT → ThingsBoard — Your Device on a Dashboard
**Day 2 · ~2 hours · Participant guide**

## What you'll build
**In plain words:** a room monitor that appears on a web dashboard you can open from any browser —
live readings, a chart over time, and an alarm indicator. Then it works the other way too: type a new
comfort limit into the browser, and the device on the bench obeys it.

**The moment it works:** the dashboard goes on the projector, you breathe on the sensor, and the
whole room watches the line move. Then you change the limit from the browser and hear the buzzer
answer.

## Objective
From request/response to publish/subscribe — and onto a wall dashboard. By the end you can:
- **Explain why pub/sub exists** — polling a flaky server couples your device to its uptime; a broker decouples them.
- **Design topics** — namespaced hierarchies and wildcards, not a room-wide `temperature` collision.
- **Run `PubSubClient` properly** — `client.loop()` every iteration, non-blocking publish, backoff reconnect, LWT.
- **Authenticate to ThingsBoard** — the token goes in the MQTT **username**; the AI will invent anything but that.
- **Close the loop** — a dashboard RPC changes your device's threshold from a browser.

## Knowledge you'll learn first (in this order)
1. **Why pub/sub** — decoupled publishers, a broker in the middle, and no polling. Compare it with request/response and note what each costs you.
2. **Topics** — hierarchy, wildcards (`+`, `#`), and why flat topic names don't scale past one device.
3. **QoS, retain, LWT** — at-most-once vs at-least-once; retained messages; the **Last Will** that tells the world your device died.
4. **`PubSubClient`** — `setServer`, `connect`, `publish`, `subscribe`, callback, and the **`loop()` you must call**.
5. **Keepalive** — what actually keeps the session up, and why a blocking loop drops you off the broker.
6. **ThingsBoard's device API** — telemetry on `v1/devices/me/telemetry`, and **the access token goes in the MQTT username**.
7. **RPC** — how the dashboard sends a command down to the device.

## Hardware this topic
| Role | Part | Where |
|---|---|---|
| Carrier + WiFi | ROBO ESP32 | — |
| Display | OLED SSD1315 | I²C `0x3C`, Grove Port 2 (D21/D22) |
| Environment | Crowtail DHT11 | digital 1-wire |
| Local threshold | Rotary Angle | ADC1 (D32/D33) |
| Alarm | Buzzer + NeoPixel | D23 / D15 |
| Broker | HiveMQ public / local Mosquitto | network |
| Cloud | `demo.thingsboard.io` or local | network |

## Your requirement
Extend your monitor so that:
- Telemetry publishes **over MQTT** on a namespaced topic, non-blocking.
- The device appears on a **live ThingsBoard dashboard** — values, chart, alarm indicator.
- A **cloud RPC changes the threshold**; knob / HTTP / cloud precedence is defined and the OLED shows the active source.
- The **access token lives in NVS** — not in source, not in the boot banner.
- Broker loss → local monitoring continues; reconnect with backoff; **LWT** announces the device is gone.

> **How you capture this:** new PlatformIO project (you create it, toolchain proven first — `framework/START_PROMPT.md` §0), starter in, kick off with `framework/START_PROMPT.md`. Spend the interview on the broker,
> topic design, token handling and the downlink command.

### Starter interview — suggested answers (T6)
| Area | Your answer |
|---|---|
| Problem | The monitor publishes to a broker and lands on a live ThingsBoard dashboard; the cloud can adjust it back. |
| Users *(opt.)* | Facilities operator watching a wall dashboard, not serial consoles. |
| Behaviour | Publish JSON every 5–10 s; subscribe for commands; validated RPC threshold applied; OLED shows KNOB/HTTP/CLOUD source. |
| Hardware | ROBO ESP32 (onboard WiFi) + DHT11 + OLED SSD1315 + Rotary Angle; onboard buzzer and NeoPixel. |
| Documents | The DHT11, OLED and Rotary spec cards + board datasheet, plus the broker host and your ThingsBoard device token. |
| Interfaces | TB telemetry `v1/devices/me/telemetry`; RPC `v1/devices/me/rpc/request/+`; own broker `training/<room>/<device>/telemetry`. |
| Connectivity | MQTT to the room broker + ThingsBoard. **Auth: token = MQTT username, password empty.** |
| Constraints | `client.loop()` every iteration; non-blocking publish + backoff reconnect; token in NVS; namespaced topics; tiny callback; validate every inbound JSON payload before acting on it. |
| Safety | **Low risk, but the cloud can now command the device.** Range-check every RPC before applying it; an out-of-range value is rejected and logged, never applied. Losing the broker must not leave the alarm disabled — local alarming continues regardless. |
| Failure modes | Broker gone → local alarm continues, backoff reconnect, LWT fires; malformed command rejected, never applied. |
| Reuse *(opt.)* | Any prompt templates you already have that fit — a defensive JSON-parse template, or a non-blocking WiFi pattern. If you have none yet, this project starts the library. |
| Out of scope | No TLS/8883, no provisioning API, no OTA, no rules engine. |
| Acceptance | Live dashboard with chart + indicator; RPC changes threshold; token in NVS; survives broker drop; `client.loop()` serviced every iteration. |

## Flow (stages)
- **Stage 0 — Broker check (10 min):** confirm the broker the trainer chose and your ThingsBoard device + access token. Note the token — you'll need it in an unexpected place.
- **Stage 1 — Pub/sub by hand (20 min):** with `mosquitto_pub` / `mosquitto_sub` on your laptop, subscribe to a topic and publish to it. Watch a wildcard subscription catch messages from your neighbour. No microcontroller yet — understand the model first.
- **Stage 2 — Device publishes (25 min):** prompt the AI for `PubSubClient` publishing your readings to your own namespaced topic on a non-blocking interval. Watch them arrive in `mosquitto_sub`. Save an **mqtt-publish** template.
- **Stage 3 — Device subscribes (20 min):** subscribe to a command topic; publish a threshold from your laptop and watch the device obey it. Keep the callback tiny — same discipline as an ISR.
- **Stage 4 — ThingsBoard auth (25 min):** repoint at ThingsBoard and publish to `v1/devices/me/telemetry`. **This is where it will fail.** Fix the auth, then move the token into NVS.
- **Stage 5 — Dashboard + RPC (20 min):** build a dashboard (latest values + a time-series chart + an alarm indicator), then add an RPC control that sets the threshold from the browser. Show it on the projector.

## Catch the AI
- ⚠ **The ThingsBoard auth trap.** Ask the AI to connect to ThingsBoard MQTT and it will invent something — an `Authorization` header, a bearer token in the payload, a password field, a REST-style API key. **None of that exists in MQTT.** The access token goes in the **username**, with an **empty password**. Nothing in the error message tells you this; you get a bare connect failure. Read the ThingsBoard docs, fix it, and log the catch — this is the highest-value AI failure in the whole kit, because it fails *confidently and silently*.
- ⚠ **Flat topics.** It will publish to `temperature`. With 12 boards in the room that's a collision. Demand a namespace (`site/floor/device/metric`).
- ⚠ **Forgetting `client.loop()`** — or calling it only when publishing. The session drops on keepalive and reconnects look "random."
- ⚠ **Blocking reconnect** — `while(!client.connected()) delay(5000)` stalls every other job on the device while the broker is away.
- ⚠ **Token hardcoded** in source, and often printed to serial on boot.

## Done when (shared objective)
- [ ] Telemetry publishing over MQTT on a **namespaced** topic, non-blocking.
- [ ] Live on a **ThingsBoard dashboard**: latest values, time-series chart, alarm indicator.
- [ ] Cloud **RPC changes the threshold**; OLED shows which source is active.
- [ ] Token in NVS, not in source, not printed.
- [ ] Broker drop → device keeps monitoring locally and reconnects with backoff.
- [ ] `client.loop()` serviced every iteration.

## Save to your prompt library
- `mqtt-publish` template · `mqtt-subscribe-command` template · `thingsboard-auth` template (write down the username-token rule — the AI will get it wrong again next time).
