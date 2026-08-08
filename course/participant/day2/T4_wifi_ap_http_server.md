# T4 · WiFi Access Point + HTTP Server — Your Board Serves Data
**Day 2 · ~2 hours · Participant guide · works in pairs**

## What you'll build
**In plain words:** your board becomes its own WiFi network. You wave your hand over a sensor and it
remembers what you did. Your partner connects their laptop, phone — or their own board — to your
network with no router involved, asks your board what it last saw, and gets the answer back as data.
They can also send *their* gesture to your board, and your light shows it.

**The moment it works:** your partner joins a WiFi network that is physically sitting on your desk,
types an address into their browser, and sees the hand movement you just made appear as text.

## Objective
By the end you can:
- **Run the board as an access point** — it creates the network rather than joining one, with a known fixed address and no router in the picture.
- **Serve HTTP endpoints** — a `GET` that returns live data as JSON, and a `POST` on the same path that accepts a peer's data.
- **Keep the server off the critical path** — gesture detection and indication carry on at full speed while clients hammer the endpoints.
- **Treat every request as untrusted** — malformed bodies, unknown values, wrong methods and unknown paths all get a sane answer instead of a crash.
- **Test like a client** — drive your own API from a browser and from the command line before anyone else does.

## Knowledge you'll learn first (in this order)
1. **AP vs station mode** — `WiFi.softAP()` creates a network; `WiFi.begin()` joins one. They are different modes and the AI mixes them up.
2. **You are the DHCP server** — in AP mode the board does not *get* an address, it *hands them out*. Its own address is fixed and predictable (`192.168.4.1` by default).
3. **The `WebServer` library** — `on()` to register routes, `begin()` to start, and the `handleClient()` you must call every loop.
4. **Routes and methods** — the same path with `GET` and `POST` are two entirely different handlers.
5. **Reading a request body** — `server.arg("plain")` for a raw JSON POST, and why query parameters are not the same thing.
6. **Response codes and content types** — `200` with `application/json`, `400` for a bad body, `404` for an unknown path.
7. **Handlers must be fast** — anything slow inside a handler stalls every other request and your own device.
8. **Events, not levels** — a gesture is a discrete thing that *happened*. You latch it, serve it, and decide how long it stays current.

## Hardware this topic
| Role | Part | Interface | Where |
|---|---|---|---|
| Carrier + WiFi | ROBO ESP32 (onboard radio, AP mode) | — | — |
| Input | **Grove Gesture PAJ7660** | I²C **`0x73` — verify by scan** | Grove Port 2 (D21/D22) |
| Indication | NeoPixel + buzzer | digital | D15 / D23 |
| Client | your partner's laptop, phone or board | WiFi | joins your AP |

> **You work in pairs.** Each of you builds and runs your own server; your partner is your client.
> Agree distinct SSIDs so you are not both broadcasting the same name.
>
> **No display this topic.** The gesture sensor takes the board's single I²C port, so status goes to
> the serial monitor and the NeoPixel. That is normal for a headless field device.
>
> **If your gesture module misbehaves**, fall back to the onboard buttons (D34/D35) as the event
> source — press instead of wave. Nothing else in this project changes.

## The shared contract — both boards implement this

Every server in the room serves **exactly this**, so **any client can talk to any server**. Agree
nothing; it is already agreed. That is the point — interoperability comes from a written contract,
not from a conversation between two people who happen to be sitting together.

```
GET /api/gesture
  -> 200 application/json
     {"device":"node-a","gesture":"SWIPE_LEFT","count":7,"age_ms":420,"uptime_s":1234}

POST /api/gesture
  body: {"device":"node-b","gesture":"SWIPE_UP"}
  -> 200 {"ok":true}
  -> 400 {"ok":false,"error":"<reason>"}   malformed body or unknown gesture
  -> 404                                    any other path
```

**Accepted gestures** — anything else is rejected with `400`, never stored, never shown:
`SWIPE_LEFT` · `SWIPE_RIGHT` · `SWIPE_UP` · `SWIPE_DOWN` · `NONE`.
`device` must be a non-empty string. `count` and `uptime_s` are unsigned; `age_ms` is how long ago
the gesture was seen.

**One path, two methods, two different handlers.** `GET` returns the serving board's own latest
gesture. `POST` accepts another board's gesture, which the server shows on its NeoPixel.

## Your requirement
Build a **self-contained gesture server**, where:
- The board **creates its own WiFi network** with a named SSID and a password — no router involved.
- It detects gestures, **latches the most recent one** with a count and an age, and shows it on the NeoPixel.
- It serves **`GET /api/gesture`** and **`POST /api/gesture`** exactly as the contract above states.
- **Gesture detection and indication keep running at full rate** while requests are being served.
- A malformed body, an unknown gesture, a wrong method or an unknown path returns a proper code — never a crash, never a hang.
- **Any client following the contract works**, including a board someone else built.

> **How you capture this:** create a **new PlatformIO project** (you create it, toolchain proven first —
> `framework/START_PROMPT.md` §0), copy `framework/project_starter.json` into it, and kick off with
> `framework/START_PROMPT.md`. Spend the interview on the endpoint contract and on what an untrusted
> request may and may not change.

### Starter interview — suggested answers (T4)
| Area | Your answer |
|---|---|
| Problem | A touchless input node that runs its own WiFi network and serves what it last saw over HTTP, so a phone, laptop or another board can read it with no infrastructure at all. |
| Users *(opt.)* | A technician on site with a phone and no network available — connects directly to the device. |
| Behaviour | Board starts an AP at boot. Gesture detected → latched with a count and timestamp → shown on the NeoPixel. `GET /api/gesture` returns the latest as JSON. `POST /api/gesture` accepts a peer's gesture and flashes it. Detection and indication run continuously regardless of traffic. |
| Hardware | ROBO ESP32 (onboard WiFi in AP mode) + Grove Gesture PAJ7660 on the single I²C port; onboard NeoPixel and buzzer. No display. |
| Documents | Attach `hardware/modules/Grove-Gesture_sensor_paj7660.md` and the board datasheet. |
| Interfaces | Gesture = I²C on **Grove Port 2 (D21/D22)**, address **`0x73` — verify by scanning, the vendor doc is not definitive**. NeoPixel D15, buzzer D23. |
| Connectivity | **Access-point mode only** — the board creates the network. Fixed IP `192.168.4.1`, HTTP server on port 80. Serves `GET` and `POST` on `/api/gesture` per the shared contract. No internet, no router, no cloud. |
| Constraints | `server.handleClient()` every loop iteration; handlers must return immediately — never read the sensor inside a handler, serve the latched value; gesture polling on its own timer; gestures debounced and rate-limited so one hand movement is one event; AP password set, not an open network; no `delay()` in the loop. |
| Safety | Sensing and indication only — nothing moves or heats. **Must never happen:** an inbound request changing what the device shows or does without validation, or HTTP traffic delaying gesture detection. **Safe state on boot:** AP up, NeoPixel off, latched gesture `NONE` — before any client can connect. A `POST` carrying an unknown gesture string or a malformed body is rejected with `400`; the last good value is retained and nothing is shown from the bad request. |
| Failure modes | Malformed or non-JSON body → `400`, nothing stored. Unknown gesture string → `400`. Unknown path → `404`. Wrong method on a known path → `405` or `404`, never handled anyway. Gesture sensor not found on the bus at boot → say so on serial and keep serving `NONE` rather than hanging. No clients connected is normal, not an error. |
| Reuse *(opt.)* | Any prompt templates you already have that fit — non-blocking timers, I²C device setup, NeoPixel indication. If you have none yet, this project starts the library. |
| Out of scope | No station mode, no internet, no HTTPS, no authentication beyond the AP password, no serving HTML pages — JSON only. No endpoints beyond the two in the contract. |
| Acceptance | Partner joins the AP and gets contract-valid JSON from `GET /api/gesture` that changes when you wave; a valid `POST /api/gesture` visibly flashes your NeoPixel; an unknown gesture or malformed body is rejected with `400` and changes nothing; an unknown path returns `404`; detection and indication keep pace while the endpoints are hammered; **a client built by someone outside your pair works without changes.** |

## Flow (stages)
- **Stage 1 — Prove the sensor (20 min):** scan the I²C bus and **confirm `0x73` appears** before writing any driver code. Get gestures printing to serial and flashing the NeoPixel — **no network code at all yet**. **Test:** four distinct swipes produce four distinct, debounced serial lines.
- **Stage 2 — Become a network (20 min):** prompt for `WiFi.softAP()` with your own SSID and password; print SSID and IP to serial. **Test:** your partner sees your network in their WiFi list and joins it; `ping 192.168.4.1` answers. Save a **wifi-softap** template.
- **Stage 3 — Serve the gesture (25 min):** add `GET /api/gesture` returning the contract JSON. **Test:** partner opens `http://192.168.4.1/api/gesture` in a browser, you wave, they refresh — the gesture and count change. Save a **http-server-get** template.
- **Stage 4 — Accept a peer gesture (25 min):** add `POST /api/gesture` that parses the body, validates the gesture against the allowed list, and flashes the NeoPixel. **Test:** partner posts `SWIPE_UP` with `curl` and your light responds; then they post `"gesture":"BANANA"` and it is rejected `400` with nothing shown. Save a **http-server-post** template.
- **Stage 5 — Don't stall (20 min):** confirm `handleClient()` runs every loop and the sensor is polled on its own timer, never inside a handler. **Test:** partner refreshes as fast as they can while you keep waving — every gesture still registers and the NeoPixel keeps up.
- **Stage 6 — Hostile clients, and a stranger (10 min):** partner sends a body that isn't JSON, an unknown gesture, and a request to `/nonsense`. **Test:** each returns a sensible code and the device carries on. Then **swap with a different pair** — their client should drive your server with no code changes on either side.

## Catch the AI
- ⚠ **The headline trap: it writes station mode.** Ask for "connect the ESP32 to WiFi" and you get `WiFi.begin(ssid, password)` and a wait-for-IP loop. That **joins** a network. You want `WiFi.softAP()`, which **creates** one. The AI mixes these constantly, and the symptom is a board waiting forever for a router that does not exist.
- ⚠ **Waiting for a DHCP address in AP mode.** In AP mode the board *is* the DHCP server — its address is fixed. Any code polling for an assigned IP is confused about which mode it is in.
- ⚠ **Missing `server.handleClient()`** — or calling it only sometimes. The server registers routes, reports no error, and simply never answers.
- ⚠ **Gesture address invented.** It will state an I²C address with total confidence. **Scan for `0x73`** — the vendor documentation itself flags it as needing verification.
- ⚠ **Sensor read inside a handler.** It stalls every other request and drops gestures. Poll on a timer, latch the result, serve the latch.
- ⚠ **No debounce or rate limit.** One hand movement becomes five events and the count runs away.
- ⚠ **No 404 and no method check.** Unknown paths hang or return a blank 200; a `GET` to a POST-only route gets handled anyway.
- ⚠ **Body vs query parameters.** For a raw JSON POST the body is `server.arg("plain")` — the AI often reaches for named parameters that were never sent.
- ⚠ **No validation on the POST.** It will show whatever string arrives. Check it against the allowed list and answer `400` when it fails.
- ⚠ **It invents its own JSON shape.** Ask for a gesture endpoint and it picks its own field names — `type`, `dir`, `ts`. The contract above is fixed; any deviation breaks every other client in the room.
- ⚠ **An open AP with no password**, or credentials hardcoded and printed to serial on boot.

## Done when (shared objective)
- [ ] **`0x73` confirmed by bus scan**, not assumed.
- [ ] Board creates its **own named WiFi network**; SSID and IP on serial.
- [ ] Partner joins it and gets **contract-valid JSON** from `GET /api/gesture` that **changes when you wave**.
- [ ] Partner's `POST /api/gesture` **visibly flashes your NeoPixel**.
- [ ] Unknown gesture → `400`, malformed body → `400`, unknown path → `404`, and the device carries on.
- [ ] Gesture detection keeps pace **while the endpoints are being hammered**.
- [ ] `server.handleClient()` every loop; no `delay()`; no sensor read inside a handler.
- [ ] **A client from another pair works against your server unchanged** — the contract holds.

## Save to your prompt library
- `wifi-softap` template · `http-server-get` template · `http-server-post` template · `validate-request-body` template · `i2c-scan-then-init` template.
