# T5 · WiFi Client + HTTP Client — Consuming a Server That Lies to You
**Day 2 · ~2 hours · Participant guide · works in pairs**

## What you'll build
**In plain words:** your board joins someone else's WiFi network and talks to a server on it. It asks
that server what hand gesture it last saw and mirrors it on your own light — **your partner waves,
your light changes.** It also sends your own gestures up to them. The interesting part is what
happens when the server misbehaves, because it will.

**The moment it works:** your partner waves at their board across the desk and your NeoPixel answers.
Then they pull the power out mid-request and yours carries on as if nothing happened.

## Objective
By the end you can:
- **Join a network without blocking** — a connect state machine with timeout and backoff, so nothing else on the device stalls waiting for WiFi.
- **Make HTTP requests** — a `GET` that fetches the peer's state, and a `POST` on the same path that uploads yours.
- **Parse defensively** — status code, then parse error, then field presence and type, before any value is trusted.
- **Fail toward last-good** — a broken server never leaves the device acting on garbage.
- **Set timeouts that mean something** — a server that accepts your connection and then goes silent must cost you seconds, not forever.

## Knowledge you'll learn first (in this order)
1. **Station mode** — `WiFi.begin()` joins an existing network; SSID, password, and what `WiFi.status()` actually reports.
2. **Why blocking connect is a production bug** — `while (WiFi.status() != WL_CONNECTED)` freezes everything else on the device.
3. **The connect state machine** — DISCONNECTED → CONNECTING → ONLINE on `millis()`, with a timeout and exponential backoff.
4. **`HTTPClient`** — `begin`, `GET`, `POST`, `getString`, and the `end()` you must call on every path.
5. **Status codes are data** — a negative return is a *transport* failure; `200` vs `4xx` vs `5xx` is an *application* answer. They need different handling.
6. **ArduinoJson** — `JsonDocument`, `deserializeJson`, and the `DeserializationError` you must check.
7. **Defensive field access** — `doc["x"] | fallback`, `is<T>()`, and why a missing key silently reads as `0` or an empty string.
8. **Timeouts** — `setTimeout()`, and what a silent server does to a device without one.

## Hardware this topic
| Role | Part | Interface | Where |
|---|---|---|---|
| Carrier + WiFi | ROBO ESP32 (onboard radio, station mode) | — | — |
| Input | **Grove Gesture PAJ7660** | I²C **`0x73` — verify by scan** | Grove Port 2 (D21/D22) |
| Indication | NeoPixel + buzzer | digital | D15 / D23 |
| Server | your partner's board, or the trainer's fallback server | WiFi | the network you join |

> **You work in pairs.** One board runs a server on its own access point; the other joins it and runs
> the client. **Swap roles once the client works** so you both build the client half. If your partner
> is not ready, point your client at the trainer's fallback server — the client code does not care
> which server it is talking to.
>
> **No display this topic.** The gesture sensor takes the board's single I²C port, so status goes to
> the serial monitor and the NeoPixel.
>
> **If your gesture module misbehaves**, fall back to the onboard buttons (D34/D35) as the event
> source. The client half of this project is unaffected either way.

## The shared contract — the server you talk to implements this

Every server in the room serves **exactly this**, so **your client works against any of them** —
your partner's board, another pair's board, or the fallback server. You do not negotiate it; it is
already agreed. That is the point: interoperability comes from a written contract.

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

**Accepted gestures** — the server rejects anything else, and so must you before you trust a value:
`SWIPE_LEFT` · `SWIPE_RIGHT` · `SWIPE_UP` · `SWIPE_DOWN` · `NONE`.

**You consume the `GET` and drive the `POST`.** Take the peer's `gesture` from the `GET` and mirror
it; send your own to the `POST`. **Assume nothing about the server's health** — the fields above are
what a *correct* server returns, not what you will always receive.

## Your requirement
Build a **network client node**, where:
- The board **joins an existing WiFi network** as a station, non-blocking, with backoff on failure.
- It **`GET`s `/api/gesture`** on an interval and **mirrors the peer's gesture on your NeoPixel** when the answer is valid.
- It detects your own gestures and **`POST`s them** to `/api/gesture` in the contract's shape.
- The serial output shows link state, the last peer gesture, and **whether that value is fresh or stale**.
- Any server failure — unreachable, slow, `500`, HTML instead of JSON, half a response, an unknown gesture string — means **hold the last good value, mark it stale**, and retry later.
- **No network call ever blocks** gesture detection or indication.
- **It works against any contract-compliant server**, not just your partner's.

> **How you capture this:** create a **new PlatformIO project** (you create it, toolchain proven first —
> `framework/START_PROMPT.md` §0), copy `framework/project_starter.json` into it, and kick off with
> `framework/START_PROMPT.md`. Spend the interview on the endpoint contract and on exactly what the
> device does when each part of that contract is broken.

### Starter interview — suggested answers (T5)
| Area | Your answer |
|---|---|
| Problem | A touchless node that joins a network, mirrors what a peer node saw, reports its own gestures back, and keeps working correctly when that peer misbehaves. |
| Users *(opt.)* | Two touchless control points in different rooms, one mirroring the other. |
| Behaviour | Non-blocking join. Every 1–2 s: `GET /api/gesture` and mirror the peer's gesture on the NeoPixel if valid. On each local gesture: `POST` it. Serial shows link state, peer gesture and freshness. Local detection and indication run regardless of network state. |
| Hardware | ROBO ESP32 (onboard WiFi in station mode) + Grove Gesture PAJ7660 on the single I²C port; onboard NeoPixel and buzzer. No display. |
| Documents | Attach `hardware/modules/Grove-Gesture_sensor_paj7660.md` and the board datasheet. The endpoint contract is fixed and stated above — no negotiation needed. |
| Interfaces | Gesture = I²C on **Grove Port 2 (D21/D22)**, address **`0x73` — verify by scanning**. NeoPixel D15, buzzer D23. |
| Connectivity | **Station mode** — joins the server's AP. `GET /api/gesture` to read the peer, `POST /api/gesture` to send mine. Plain HTTP, port 80, server at `192.168.4.1`. |
| Constraints | No blocking connect — state machine with timeout and backoff capped ~30 s; explicit `setTimeout()` on every request; `http.end()` on every path including failures; ArduinoJson only, never `String` concatenation; requests on their own `millis()` interval, never in the detection path; gestures debounced so one movement is one POST. |
| Safety | Sensing and indication only — nothing moves or heats. **Must never happen:** an unvalidated gesture string from the network driving local indication, or a slow/unreachable server delaying local gesture detection. Every received value is checked against the allowed list before use; anything invalid is discarded and the last good value is held and marked stale. **Safe state on boot:** NeoPixel off, peer gesture `NONE`, before any request succeeds. |
| Failure modes | Transport failure, non-200, parse error, missing or wrong-typed field, unknown gesture string → keep last-good, mark stale, retry next interval. Silent server → request times out in ≤5 s. Network lost → keep detecting and indicating locally, reconnect with backoff. |
| Reuse *(opt.)* | Any prompt templates you already have that fit — non-blocking timers, I²C device setup, NeoPixel indication. If you have none yet, this project starts the library. |
| Out of scope | No server code on this board, no MQTT, no TLS, no auth beyond the AP password, no OTA. |
| Acceptance | Joins the network without ever blocking; mirrors a valid peer gesture on the NeoPixel; survives `500`, HTML body, truncated JSON, unknown gesture and a silent server with no crash and no bad action; marks stale visibly and recovers; POSTs contract-shaped JSON the server answers `200` to; **works unchanged against another pair's server**; local detection unaffected throughout. |

## Flow (stages)
- **Stage 1 — Prove the sensor and find your server (20 min):** scan the bus, **confirm `0x73`**, get local gestures printing and flashing — **no network code yet**. Then note your server's SSID, password and IP, and confirm with `curl` from a laptop that it actually answers.
- **Stage 2 — Join without blocking (20 min):** prompt for a station connect as a state machine with timeout and backoff. **Test:** keep waving while it connects — every gesture still registers, and giving it a **wrong password** freezes nothing. Save a **wifi-station-nonblocking** template.
- **Stage 3 — First GET (20 min):** fetch `/api/gesture` and mirror the peer's gesture on your NeoPixel. **Test:** partner waves at their board, your light answers. **This is the moment.** Save an **http-client-get** template.
- **Stage 4 — Break the server on purpose (30 min):** ask your partner to make their endpoint return `500`, then an HTML page, then half a JSON object, then an unknown gesture string, then nothing at all. Harden the parse: transport → status → `DeserializationError` → field presence and type → allowed-list check → last-good fallback. **Test:** all five hostile modes handled, local detection never disturbed. **This stage is the topic.**
- **Stage 5 — POST your gestures (20 min):** send each local gesture in the contract shape on a debounced, non-blocking path. **Test:** your partner's NeoPixel flashes when *you* wave. Then **point your client at another pair's server** — it should work with no code change.
- **Stage 6 — Pull the plug (10 min):** partner powers down their board mid-request. **Test:** your request times out within 5 s, the peer value is marked stale, local gestures keep working, and it recovers by itself when their board returns.

## Catch the AI
- ⚠ **Blocking connect.** It will write `while (WiFi.status() != WL_CONNECTED) { delay(500); }`. If the network is absent that is forever, and your device is dead while it waits. Demand a state machine.
- ⚠ **The fragile parse — the headline trap.** It writes `doc["gesture"]` with no status check and no `DeserializationError` check. Ask *"what if the server returns 500, or an HTML login page?"* then actually make that happen. Its code reads an empty string and your logic acts on it.
- ⚠ **No allowed-list check.** Even a well-formed response can carry a gesture string you do not recognise. Validate against the list before it drives anything.
- ⚠ **No timeout.** The default behaviour against a server that accepts the connection then says nothing is a long stall. Demand an explicit `setTimeout()`.
- ⚠ **Treating `-1` as an HTTP status.** A negative return is a transport failure, not a response code — they need different handling.
- ⚠ **Gesture address invented.** **Scan for `0x73`** — the vendor documentation itself flags it as needing verification.
- ⚠ **`String` concatenation to build JSON** — heap fragmentation on a device meant to run for weeks. Demand ArduinoJson serialization.
- ⚠ **Missing `http.end()`**, or a fresh `begin()` every loop — connections leak until the device stops working.
- ⚠ **Assuming the server is up.** Its happy-path code has no branch for "no reply at all", which is the most common real condition.
- ⚠ **It invents its own JSON shape.** Left alone it will send `type`, `dir`, `ts` — its own field names — and the server will reject every POST with `400`. The contract is fixed; paste it into the prompt.
- ⚠ **Hardcoded SSID and password** in the source, often printed to serial at boot.

## Done when (shared objective)
- [ ] **`0x73` confirmed by bus scan**, not assumed.
- [ ] Joins the network with **no blocking loop** — local gestures keep registering throughout, including on a wrong password.
- [ ] **Partner waves, your NeoPixel answers** — a valid peer gesture is mirrored.
- [ ] **`POST` sends contract-shaped JSON** the server answers `200` to, and their light responds to your wave.
- [ ] Survives **`500`, HTML instead of JSON, truncated JSON, an unknown gesture and a silent server** — no crash, no hang, no acting on garbage.
- [ ] Holds the last good value, **marks it stale**, and recovers by itself when the server returns.
- [ ] Every request has an explicit timeout; `http.end()` on all paths; detection never blocked.
- [ ] **Works unchanged against another pair's server** — the contract holds.

## Save to your prompt library
- `wifi-station-nonblocking` template · `http-client-get` template · `http-client-post` template · `defensive-json-parse` template — the last one is the most reusable thing you will write all week.
