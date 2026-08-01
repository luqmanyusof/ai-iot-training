# T7 · UART & RS485 — Cross-Control Across an Industrial Bus
**Day 3 · ~2 hours · Participant guide**

## Objective
Drop the network entirely and build a wired industrial link. By the end you can:
- **Use UART deliberately** — three ESP32 UARTs, remapped pins, and the `Serial2.begin(baud, SERIAL_8N1, RX, TX)` pin order that silently breaks everything when reversed.
- **Understand a differential bus** — why RS485 reaches 1200 m when plain UART dies at 15, and what A/B polarity and termination actually do.
- **Solve half-duplex contention** — two nodes, one pair of wires, and only one allowed to talk at a time. This is the lesson.
- **Design a real wire protocol** — address, delimiter, length, payload, checksum. Nothing frames your bytes for you.
- **Make an actuator safe** — decide what a servo does when the bus goes quiet, before it does something you didn't intend.

## Knowledge you'll learn first (in this order)
1. **UART fundamentals** — baud, framing, TX↔RX crossover, and that UART has no addressing and no acknowledgement.
2. **ESP32 UARTs** — `Serial` is your USB console; `Serial1`/`Serial2` are yours, with remappable pins.
3. **RS485 = UART on a differential pair** — same bytes, different physical layer. Noise immunity, long runs, multi-drop.
4. **Half-duplex** — 2-wire RS485 is one lane, both directions. Two talkers at once = garbage on the wire.
5. **The turnaround problem** — `flush()`, and why "send then immediately read" loses the reply.
6. **Framing and addressing** — a stream gives you bytes, not messages, and a bus gives you everybody's bytes, not just yours.
7. **Fail-safe actuation** — hold, centre, or park? An actuator with no defined behaviour on link loss is a hazard.

## Hardware this topic
| Role | Part | Interface | Where |
|---|---|---|---|
| Bus | Grove RS485 | **UART** | UART Grove port (**D16/D17**) |
| Your input | Rotary Angle | analog | **ADC1 (D32/D33)** |
| Your output | **TS90A micro servo** (3–6 V, 3-pin lead, not Grove) | PWM | servo header (D4/D5/D18/D19) |
| Display | OLED SSD1315 | I²C `0x3C` | Grove Port 2 (D21/D22) |
| Link state | NeoPixel / buzzer | digital | D15 / D23 |
| Console | USB serial | UART0 | 115200 |

> **You work in pairs — two boards, one bus.** Join them **A→A and B→B** on a twisted pair. Both
> boards run the **same firmware** with a different `NODE_ID`.

| Pair | Node IDs |  | Pair | Node IDs |
|---|---|---|---|---|
| P1 | 01 / 02 |  | P4 | 07 / 08 |
| P2 | 03 / 04 |  | P5 | 09 / 10 |
| P3 | 05 / 06 |  | P6 | 11 / 12 |

**Same baud at both ends.** A baud mismatch and swapped A/B both produce **silence, not an error** —
keep that at the top of your debugging checklist.

## Your requirement
Build a **cross-control link** where:
- **Your knob drives your partner's servo**, and **their knob drives yours** — at the same time, both directions live.
- The servo tracks the remote knob **within ~200 ms**, smoothly, with no visible stepping.
- Every message is **addressed, framed and checksummed** — a node ignores traffic that isn't for it.
- A **corrupted frame is rejected**, not acted on. The servo must never twitch on bad data.
- **Link lost → the servo goes to a defined fail-safe position** and the OLED shows `STALE`. It must not hold a stale command forever.
- Nothing blocks: your own knob, display and bus traffic all keep running.

> **How you capture this:** create a **new PlatformIO project** (you create it, toolchain proven first —
> `framework/START_PROMPT.md` §0), copy `framework/project_starter.json` into it, and kick off with
> `framework/START_PROMPT.md`. Answer "same as \<earlier topic\>" for anything unchanged; the new
> datasheets are `hardware/modules/Grove-RS485.md` and `hardware/modules/TS90A-Micro-Servo.md`.

### Starter interview — suggested answers (T7)
| Area | Your answer |
|---|---|
| Problem | Two nodes on an RS485 bus cross-control each other: my knob moves your servo, yours moves mine. |
| Users *(opt.)* | Building automation — a remote pump house on an RS485 trunk, no network available. |
| Behaviour | Every ~100 ms: read knob → send addressed frame → partner's servo tracks it. Both directions at once. Bad/missing frames → reject, then fail-safe. |
| Hardware | ROBO ESP32 + Grove RS485 + Rotary Angle + Servo + OLED; two identical boards. |
| Documents | Attach `hardware/modules/Grove-RS485.md`, `hardware/modules/TS90A-Micro-Servo.md`, `hardware/modules/Grove-Rotary_Angle_Sensor.md` + the board datasheet (UART port, servo headers). |
| Interfaces | RS485 = **UART on D16/D17** (`Serial2`); servo = PWM on a servo header; knob = **ADC1 only**. |
| Connectivity | RS485 half-duplex 2-wire, 2 nodes, A→A / B→B, matched baud, 120 Ω terminated. |
| Constraints | **ESP32Servo**, not `Servo.h`; non-blocking servo motion; only one node may drive the bus at a time; `flush()` before switching to receive; frames addressed + checksummed; knob stays on ADC1. |
| Failure modes | Bad checksum → reject + count; wrong address → ignore; ≥3 missed frames → `STALE` + servo to fail-safe position; local knob/display unaffected. |
| Reuse *(opt.)* | T2 analog-read + moving-average templates; T3 OLED + non-blocking timers; T5's defensive-parse discipline applied to bytes. |
| Out of scope | No WiFi/MQTT, no Modbus library (write your own frame), no more than 2 nodes, no encryption. |
| Acceptance | Partner's servo follows my knob < 200 ms; both directions simultaneously; corrupt frame rejected with no servo movement; unplugging a wire drives fail-safe + `STALE`; no blocking calls. |

## Flow (stages)
- **Stage 0 — Wire the bus (10 min):** find your partner, note your `NODE_ID`, confirm RS485 is on the **UART Grove port (D16/D17)** — not I²C. Join **A→A, B→B**. Check the module for a termination pad and a direction (DE/RE) pin — **look at the board, don't assume**.
- **Stage 1 — UART alone (20 min):** prompt for `Serial2` on D16/D17, jumper TX→RX, echo bytes. Prove the port and pin order **before** a bus is involved. Debug pin order here, not later.
- **Stage 2 — One direction (20 min):** Board A reads its knob and sends the value; Board B prints it to serial. Simplex only. Save a **rs485-uart-send** template.
- **Stage 3 — It moves (25 min):** Board B drives its **servo** from the received value. Turn A's knob — B's servo tracks. First "it's alive" moment. Use **ESP32Servo** and keep the motion non-blocking.
- **Stage 4 — Both directions, and the collision (25 min):** now make B send too. **It will break** — two nodes driving one pair at once produces garbage and twitching servos. Fix it properly: agree a turnaround scheme (master polls / strict time slots), add **addressing** so each node ignores traffic not meant for it, and a **checksum** so garbage never reaches the servo. This stage is the topic.
- **Stage 5 — Break it safely (20 min):** corrupt a byte on purpose → servo must not move. Unplug a bus wire → `STALE` + servo to fail-safe. Swap A/B on one end → observe silence, no error. Discuss what termination is doing on a long trunk.

## Catch the AI
- ⚠ **The headline trap: the AI treats RS485 as plain UART.** It will hand you `Serial2.print()` / `readString()` with no notion that the wire is shared. With both boards transmitting, you get garbled frames and jittering servos — and the AI's diagnosis will be "add a delay." The real answer is a **turnaround scheme**, and it has to come from you.
- ⚠ **No `flush()` before switching to receive** — you drop your partner's reply and blame the wiring.
- ⚠ **No framing.** `readString()` and hope; a message splits mid-number and your servo takes a wild angle from half a value.
- ⚠ **No addressing.** The node acts on frames meant for the other one — or on its own traffic.
- ⚠ **No checksum.** On a noisy differential line a flipped bit becomes a real servo command. This is visible, physical, and the best argument for integrity checking you'll get all week.
- ⚠ **Stock `Servo.h`.** Does not compile on ESP32 — use **ESP32Servo**. The AI still recommends the wrong one.
- ⚠ **Blocking servo sweep** — `for` loop with `delay(15)` per degree stalls your bus reads for a second.
- ⚠ **No fail-safe.** Ask it what the servo should do when frames stop arriving. It usually has no answer; "hold the last command forever" is a decision you must make consciously, not inherit by accident.
- ⚠ **Invented direction pin** (or a missing one) — the AI may add DE/RE toggling for a transceiver that auto-directs, or omit it for one that doesn't. Check the module.

## Done when (shared objective)
- [ ] RS485 confirmed on **UART D16/D17**; A→A / B→B verified.
- [ ] Knob → **partner's servo** tracks within ~200 ms, smoothly.
- [ ] **Both directions work simultaneously** without garbling the bus.
- [ ] Frames are **addressed + checksummed**; wrong-address traffic ignored.
- [ ] A deliberately corrupted frame is **rejected — the servo does not move**.
- [ ] Bus unplugged → `STALE` on the OLED and servo at its **defined fail-safe position**.
- [ ] **ESP32Servo**, non-blocking motion, no `delay()` anywhere.

## Save to your prompt library
- `uart-hardware-serial` template · `rs485-half-duplex` template · `framed-addressed-packet` template · `servo-esp32-nonblocking` template.

The framed-addressed-packet template is the one you'll reuse for the rest of your career — it applies
to Modbus, GPS, modems, any serial link you ever touch.
