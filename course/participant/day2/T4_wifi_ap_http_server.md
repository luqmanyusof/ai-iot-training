# T4 · WiFi Access Point + HTTP Server — Your Board Serves Data
**Day 2 · ~2 hours · Participant guide · works in pairs**

## What you'll build
**In plain words:** your board becomes its own WiFi network. Your partner connects their laptop,
phone — or their own board — to it, with no router and no room WiFi involved. They ask it for your
live sensor readings and get them back as data. They can also send *their* readings to your board,
and yours displays them.

**The moment it works:** your partner joins a WiFi network that is physically sitting on your desk,
types an address into their browser, and sees your temperature reading appear.

## Objective
By the end you can:
- **Run the board as an access point** — it creates the network rather than joining one, with a known fixed address and no router in the picture.
- **Serve HTTP endpoints** — a `GET` that returns live data as JSON, and a `POST` on the same path that accepts a peer's data.
- **Keep the server off the critical path** — sensing, display and alarm carry on at full speed while clients are hammering the endpoints.
- **Treat every request as untrusted** — malformed bodies, wrong content types and unknown paths all get a sane answer instead of a crash.
- **Test like a client** — drive your own API from a browser and from the command line before anyone else does.

## Knowledge you'll learn first (in this order)
1. **AP vs station mode** — `WiFi.softAP()` creates a network; `WiFi.begin()` joins one. They are different modes and the AI mixes them up.
2. **You are the DHCP server** — in AP mode the board does not *get* an address, it *hands them out*. Its own address is fixed and predictable (`192.168.4.1` by default).
3. **The `WebServer` library** — `on()` to register routes, `begin()` to start, and the `handleClient()` you must call every loop.
4. **Routes and methods** — the same path with `GET` and `POST` are two entirely different handlers.
5. **Reading a request body** — `server.arg("plain")` for a raw JSON POST, and why query parameters are not the same thing.
6. **Response codes and content types** — `200` with `application/json`, `400` for a bad body, `404` for an unknown path.
7. **Handlers must be fast** — a slow sensor read inside a handler stalls every other request and your own display.

## Hardware this topic
| Role | Part | Interface | Where |
|---|---|---|---|
| Carrier + WiFi | ROBO ESP32 (onboard radio, AP mode) | — | — |
| Environment | Crowtail DHT11 | digital 1-wire | **Grove Port 3 (D26 / D25)** |
| Display | OLED SSD1315 | I²C `0x3C` | Grove Port 2 (D21/D22) |
| Local setting | Rotary Angle | analog | **ADC1 (D32/D33)** |
| Indication | Buzzer + NeoPixel | digital | D23 / D15 |
| Client | your partner's laptop or phone | WiFi | joins your AP |

> **You work in pairs.** Each of you builds and runs your own server. Your partner is your client —
> they join your network and exercise your endpoints, and you do the same to theirs. Agree distinct
> SSIDs so you are not both broadcasting the same name.

## The shared contract — both boards implement this

Every server in the room serves **exactly this**, so **any client can talk to any server**. Agree
nothing; it is already agreed. That is the point — interoperability comes from a written contract,
not from a conversation between two people who happen to be sitting together.

```
GET /api/reading
  -> 200 application/json
     {"device":"node-a","temp":24.5,"humidity":61.0,"uptime_s":1234,"threshold":26.0}

POST /api/reading
  body: {"device":"node-b","temp":23.1,"humidity":58.0,"uptime_s":99}
  -> 200 {"ok":true}
  -> 400 {"ok":false,"error":"<reason>"}   malformed body or value out of range
  -> 404                                    any other path
```

**Accepted ranges** — anything outside these is rejected with `400`, never stored, never displayed:
`temp` −40…80 °C · `humidity` 0…100 % · `threshold` 0…50 °C · `device` a non-empty string.

**One path, two methods, two different handlers.** `GET` returns the serving board's own data and
the threshold it publishes. `POST` accepts another board's reading, which the server displays as its
peer value.

## Your requirement
Build a **self-contained sensor server**, where:
- The board **creates its own WiFi network** with a named SSID and a password — no router involved.
- It serves **`GET /api/reading`** and **`POST /api/reading`** exactly as the contract above states.
- The OLED shows the AP name, the board's IP, its own reading, the last peer reading received, and how many requests it has served.
- **The sensor, display and alarm keep running at full rate** while requests are being served.
- A malformed body, an out-of-range value, a wrong method or an unknown path returns a proper code — never a crash, never a hang.
- **Any client following the contract works**, including a board someone else built.

> **How you capture this:** create a **new PlatformIO project** (you create it, toolchain proven first —
> `framework/START_PROMPT.md` §0), copy `framework/project_starter.json` into it, and kick off with
> `framework/START_PROMPT.md`. Spend the interview on the endpoint contract and on what an untrusted
> request may and may not change.

### Starter interview — suggested answers (T4)
| Area | Your answer |
|---|---|
| Problem | A sensor node that runs its own WiFi network and serves its readings over HTTP, so a phone or laptop can read it with no infrastructure at all. |
| Users *(opt.)* | A technician on site with a phone, no network available — connects directly to the device. |
| Behaviour | Board starts an AP at boot. `GET /api/reading` returns its own temp/humidity/uptime/threshold as JSON. `POST /api/reading` accepts a peer's reading and displays it. OLED shows SSID, IP, own reading, last peer reading and request count. Sensing and alarm run continuously regardless of traffic. |
| Hardware | ROBO ESP32 (onboard WiFi in AP mode) + DHT11 + OLED SSD1315 + Rotary Angle; onboard buzzer and NeoPixel. |
| Documents | Attach the DHT11, OLED and Rotary spec cards from `hardware/modules/`, plus the board datasheet. |
| Interfaces | DHT11 = digital 1-wire on **Grove Port 3 (D26 or D25)**; OLED = I²C `0x3C` on D21/D22; knob = **ADC1 (D32/D33) only**. |
| Connectivity | **Access-point mode only** — the board creates the network. Fixed IP `192.168.4.1`, HTTP server on port 80. Serves `GET` and `POST` on `/api/reading` per the shared contract. No internet, no router, no cloud. |
| Constraints | `server.handleClient()` every loop iteration; handlers must return quickly — never read the sensor inside a handler, serve the last cached reading; sensor and display keep their own `millis()` timers; AP password set, not an open network; no `delay()` in the loop. |
| Safety | Sensing and indication only — nothing moves or heats. **Must never happen:** an inbound request changing what the device displays or does without validation, or HTTP traffic delaying the sensing and alarm path. **Safe state on boot:** AP up, alarm armed on the local knob threshold, own reading shown, peer value blank — before any client can connect. A `POST` field outside the contract ranges is rejected with `400`; the last good peer value is retained and nothing is displayed from the bad request. |
| Failure modes | Malformed or non-JSON body → `400`, nothing stored. Out-of-range field → `400`, last good peer value retained. Unknown path → `404`. Wrong method on a known path → `405` or `404`, never handled anyway. Sensor read failure → serve the last good value flagged as stale, never a fabricated zero. No clients connected is normal, not an error. |
| Reuse *(opt.)* | Any prompt templates you already have that fit — non-blocking timers, sensor reads, display rendering. If you have none yet, this project starts the library. |
| Out of scope | No station mode, no internet, no HTTPS, no authentication beyond the AP password, no serving HTML pages — JSON only. No endpoints beyond the two in the contract. |
| Acceptance | Partner joins the AP and gets contract-valid JSON from `GET /api/reading`; a valid `POST /api/reading` visibly updates the peer value on the OLED; an out-of-range or malformed POST is rejected with `400` and changes nothing; an unknown path returns `404`; display and alarm keep running while the endpoints are hammered; **a client built by someone outside your pair works without changes.** |

## Flow (stages)
- **Stage 0 — Local baseline (15 min):** sensor, display, knob and alarm working with **no network code at all**. Confirm it is solid first — everything after this is layered on top of a known-good device.
- **Stage 1 — Become a network (20 min):** prompt for `WiFi.softAP()` with your own SSID and password. Show the SSID and IP on the OLED. **Test:** your partner sees your network in their WiFi list and joins it; `ping 192.168.4.1` answers.
- **Stage 2 — Serve a reading (25 min):** add `GET /api/reading` returning the contract JSON. **Test:** partner opens `http://192.168.4.1/api/reading` in a browser and sees live values in the agreed shape; breathe on your sensor and refresh — the numbers move. Save a **http-server-get** template.
- **Stage 3 — Accept a peer reading (25 min):** add `POST /api/reading` that parses the body, range-checks every field, and displays it as the peer value. **Test:** partner posts a reading with `curl` and your OLED shows it; then they post `"temp":999` and it is rejected with `400` and nothing changes. Save a **http-server-post** template.
- **Stage 4 — Don't stall (20 min):** move the sensor read out of the handler and serve a cached value; confirm `handleClient()` runs every loop. **Test:** partner refreshes as fast as they can while you watch the display — it must keep updating smoothly and the alarm must still fire on time.
- **Stage 5 — Hostile clients, and a stranger (15 min):** partner sends a body that isn't JSON, an out-of-range value, and a request to `/nonsense`. **Test:** each returns a sensible code and the device carries on unchanged. Then **swap with a different pair** and have their client hit your server — if you both followed the contract it just works, with no code changes on either side.

## Catch the AI
- ⚠ **The headline trap: it writes station mode.** Ask for "connect the ESP32 to WiFi" and you get `WiFi.begin(ssid, password)` and a wait-for-IP loop. That **joins** a network. You want `WiFi.softAP()`, which **creates** one. The AI mixes these constantly, and the symptom is a board waiting forever for a router that does not exist.
- ⚠ **Waiting for a DHCP address in AP mode.** In AP mode the board *is* the DHCP server — its address is fixed. Any code polling for an assigned IP is confused about which mode it is in.
- ⚠ **Missing `server.handleClient()`** — or calling it only sometimes. The server registers routes, reports no error, and simply never answers.
- ⚠ **Slow work inside a handler.** A DHT11 read takes long enough to stall every other request and your own display. Serve a cached reading; do the sensing on its own timer.
- ⚠ **No 404 and no method check.** Unknown paths hang or return a blank 200; a `GET` to a POST-only route gets handled anyway.
- ⚠ **Body vs query parameters.** For a raw JSON POST the body is `server.arg("plain")` — the AI often reaches for named parameters that were never sent.
- ⚠ **No validation on the POST.** It will display whatever arrives, including nonsense. Range-check every field against the contract and answer `400` when it fails.
- ⚠ **It invents its own JSON shape.** Ask for a sensor endpoint and it will pick its own field names — `temperature`, `hum`, `ts`. The contract above is fixed; any deviation breaks every other client in the room.
- ⚠ **An open AP with no password**, or credentials hardcoded and printed to serial on boot.

## Done when (shared objective)
- [ ] Board creates its **own named WiFi network**; SSID and IP shown on the OLED.
- [ ] Partner joins it and gets **valid JSON** from `GET /api/reading` with live values.
- [ ] Partner's `POST /api/reading` **visibly updates the peer value** on your display.
- [ ] Malformed body → `400`, out-of-range value → rejected, unknown path → `404`, and the device carries on.
- [ ] **A client from another pair works against your server unchanged** — the contract holds.
- [ ] Sensing, display and alarm keep full pace **while the endpoints are being hammered**.
- [ ] `server.handleClient()` runs every loop; no `delay()` in the loop; no sensor read inside a handler.

## Save to your prompt library
- `wifi-softap` template · `http-server-get` template · `http-server-post` template · `validate-request-body` template.
