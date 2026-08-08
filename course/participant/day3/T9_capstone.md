# T9 · Capstone — Your Mini Project
**Day 3 · ~2 hours + presentations · Participant guide**

## What you'll build
**In plain words:** whatever you decide. You already have every input, every output and a working
cloud link — today you pick a device worth building, cut it down to what actually fits in two hours,
build it, and then break it on purpose to show it fails safely.

Six starting points are below — a vent controller, a cold-chain alarm, a touchless access panel, a
two-board sensing cell, a calibration rig — or bring your own idea.

**The moment it works:** you demo it, then deliberately unplug something in front of people and the
device does something sensible instead of something alarming.

## Objective
No new hardware. No new protocol. You have every input, every output and a working cloud pipeline —
today you decide **what to build with them** and defend it. By the end you can:
- **Scope a build yourself** — pick a project, cut it to fit two hours, and justify what you cut.
- **Compose what you already own** — inputs, outputs and ThingsBoard into one coherent device.
- **Harden it** — secrets audit, validated inputs, defined fail-safe states.
- **Break it on purpose** — induced failures with documented, safe outcomes.
- **Defend a build you didn't hand-write** — architecture, prompt library, and your caught-AI stories with proof.

## What you have to work with
The full kit is available. Pick what your project actually needs — nothing more.

| | Part | Notes |
|---|---|---|
| **Inputs** | Rotary Angle | analog, **ADC1 (D32/D33)** |
| | Buttons ×2 | **D34/D35, input-only** |
| | DHT11 | digital 1-wire, ~1 Hz |
| | Gesture PAJ7660 | I²C `0x73`, **takes Grove Port 2** |
| **Outputs** | NeoPixel | D15 |
| | Buzzer | D23 |
| | TS90A servo | PWM, servo header, **ESP32Servo** |
| | OLED SSD1315 | I²C `0x3C` — **only if you skip the gesture sensor** |
| **Comms** | WiFi · HTTP/JSON · MQTT → ThingsBoard | telemetry up, RPC down |
| | RS485 | two-node bus, framed + checksummed |
| **Structure** | FreeRTOS tasks, queues, I²C mutex | |

> ⚠ **One I²C port.** The board has a single I²C Grove port (Port 2). **Gesture `0x73` or OLED `0x3C` —
> pick one.** If you choose the gesture sensor, your device is headless and the ThingsBoard dashboard
> is your display. That is a legitimate architecture, and it is how most field devices actually work.

---

## Pick a project

Choose **one**. Each uses input + output + ThingsBoard, and each has a genuine engineering problem in
it. Or propose your own — same rules, get it agreed first.

### 1 · Smart Vent Controller
**Pitch:** Comfort state drives a physical damper; facilities can override from the dashboard.
- **In:** DHT11 + knob (local setpoint) · gesture (occupant override)
- **Out:** servo (vent position 0–100%) · NeoPixel (state) · buzzer (alarm)
- **Cloud:** telemetry (temp, humidity, vent %, mode) · RPC sets setpoint or forces vent position
- **The hard part:** *precedence.* Four things want to control that vent — knob, gesture, cloud, and the automatic comfort loop. Which wins, for how long, and how does the operator know which mode is active? An override with no timeout is a bug.
- **Stretch:** occupant override expires after N minutes and returns to auto.

### 2 · Cold-Chain / Excursion Monitor
**Pitch:** Log temperature, latch an alarm on excursion, require an acknowledged reset.
- **In:** DHT11 · knob (threshold) · gesture or button (acknowledge)
- **Out:** buzzer + NeoPixel (latched alarm) · servo as a physical flag
- **Cloud:** telemetry + excursion events · RPC to remotely acknowledge
- **The hard part:** *latching and acknowledgement.* The alarm must persist after the temperature recovers — silence is not the same as resolved. Who may clear it, and what does the audit trail look like? Also: what happens to a latched alarm across a reboot?
- **Stretch:** record excursion duration and peak, publish both on clear.

### 3 · Touchless Access / Arming Panel
**Pitch:** A gesture sequence arms and disarms; a servo drives the latch; the cloud sees every attempt.
- **In:** gesture (sequence, e.g. up-up-left) · button (panic)
- **Out:** servo (latch) · NeoPixel (armed/disarmed) · buzzer (feedback)
- **Cloud:** telemetry of every attempt (success/failure) · RPC to remotely lock or unlock
- **The hard part:** *security posture on an insecure device.* A gesture sequence is a weak secret, the MQTT leg is plaintext, and the RPC has no authentication. Name every one of those honestly and state your mitigation. Also: lockout after N failures, and what the latch does on power loss.
- **Stretch:** rate-limit attempts; publish a lockout event.

### 4 · Distributed Two-Node Cell *(pairs)*
**Pitch:** One board senses, the other actuates, connected by RS485 — and only one talks to the cloud.
- **In:** node A: DHT11 + knob · node B: gesture
- **Out:** node B: servo + NeoPixel
- **Cloud:** node A is the **gateway** — publishes both nodes' data, routes RPCs down the bus
- **The hard part:** *you are building a protocol gateway.* Cloud commands must be translated into RS485 frames and acknowledged. What happens when the bus is up but WiFi is down — or the reverse? Each node needs its own fail-safe.
- **Stretch:** the gateway reports the remote node's health as a separate ThingsBoard device.

### 5 · Remote Calibration Rig
**Pitch:** The cloud commands a target position, the servo moves, and the device reports the error.
- **In:** knob (manual jog) · gesture (cycle mode: jog / auto / hold)
- **Out:** servo (positioner) · NeoPixel (mode) · buzzer (out-of-range)
- **Cloud:** telemetry of commanded vs achieved position and mode · RPC sets target
- **The hard part:** *validating commands against physical limits.* A servo has end-stops your code must know about. An out-of-range command is rejected and reported — never silently clamped, never driven into a stall. And with no position feedback, how do you know it actually got there?
- **Stretch:** sweep test that reports achieved range on demand.

### 6 · Propose your own
Same rules: **≥1 input, ≥1 output, ThingsBoard telemetry + a downlink command, one genuine hard part.**
Agree it with the trainer in Stage 0 before you build.

---

## Rules that apply whatever you pick

- **≥1 input, ≥1 output, ThingsBoard telemetry, and one command coming *down* from the cloud.**
- **Every external input validated** — sensor reads, RPC payloads, RS485 frames. Out-of-range actuator commands are **rejected and logged**, not silently clamped.
- **Defined fail-safe** per failure: sensor dead, WiFi down, broker down, actuator jammed.
- **No secrets in source**, the boot banner, or git history. Namespaced topics with device identity.
- **FreeRTOS structure** — tasks, queues, I²C mutex, measured stacks.
- **ESP32Servo**, non-blocking motion. **Knob on ADC1.** No `delay()`.

> **How you capture this:** create a **new PlatformIO project** (`framework/START_PROMPT.md` §0), copy
> `framework/project_starter.json` in, kick off with the prompt. This run should be fast — you have
> nine `REQUIREMENTS.md` files to draw on and no new datasheets to read. Spend the time on **scope**
> and **failure modes**, not on re-describing hardware.

### Starter interview — suggested answers (T9)
| Area | Your answer |
|---|---|
| Problem | *(your chosen project, in one sentence)* |
| Users *(opt.)* | Who operates it locally, and who watches the dashboard. |
| Behaviour | The control loop, the override paths, and which source wins. |
| Hardware | Only what your project needs from the kit — name each part. |
| Documents | Prior `REQUIREMENTS.md` files cover everything. **No new datasheets this topic.** |
| Interfaces | One I²C device only (gesture `0x73` **or** OLED `0x3C`); servo on the servo header; knob on ADC1. |
| Connectivity | WiFi + MQTT/ThingsBoard: telemetry up, validated RPC down. RS485 only for project 4. |
| Constraints | All prior constraints still hold + validate every external input + defined fail-safe per subsystem. |
| Safety | **Required whatever you pick.** If your project moves a servo or drives a latch: define the park position for boot, fault *and* power loss; enforce travel limits in software; accept only validated commands. If it is sensing/indication only, say so and state why. Anything left running unattended needs a timeout or watchdog. |
| Failure modes | Sensor dead / WiFi down / broker down / actuator jammed → each with a **documented safe state**. |
| Reuse *(opt.)* | The entire prompt library — this is where it pays off. |
| Out of scope | **Your call — cutting scope deliberately is the engineering.** State plainly what you cut and why. |
| Acceptance | Your own testable checks, including the induced failures. |

## Flow (stages)
- **Stage 0 — Choose and scope (25 min):** pick your project, run the interview, **cut it to fit two hours**. Get REQUIREMENTS.md + PHASES.md approved. Deliberate descoping is the skill here — sprawl is not.
- **Stage 1 — Architecture (15 min):** one page — tasks, queues, where untrusted input is validated, what the fail-safe states are. You present from this.
- **Stage 2 — Build (45 min):** phase by phase, reusing your prompt library. Nothing here is new hardware, so this should be the fastest build of the week.
- **Stage 3 — Harden (20 min):** secrets audit (source, banner, git history), validate every input, write down the transport gaps you are knowingly leaving open.
- **Stage 4 — Break it & prep (15 min):** induce your failures, confirm each degrades safely, and prepare the demo.

## Present your build
Ten minutes, in this order:
1. **Demo it working** — then **break it on purpose** and show it failing safe.
2. **Architecture page** — tasks, queues, where untrusted input is validated.
3. **Prompt library** — which templates you reused today **without editing**.
4. **"Caught the AI wrong"** — your best two or three, each with the datasheet line that proved it.
5. **Security posture** — what's protected, what isn't, what you'd do with another week.

## Catch the AI
- ⚠ **Plaintext credentials.** Ask for a "complete IoT firmware" and it puts SSID, password and token straight into `main.cpp` — after a week of you moving them out. Check the boot banner too.
- ⚠ **Flat topics / no device identity** — a payload any board could have published.
- ⚠ **It will add an OLED you did not ask for.** Display code is the default shape of an example, and it will write it for a device that isn't on your bus. One I²C port — you chose which device has it.
- ⚠ **Trusting the cloud command.** An out-of-range RPC driving the servo to a physical limit. Validate, reject, log — do not silently clamp.
- ⚠ **No fail-safe state.** Ask what the actuator should do when the sensor stops responding. It usually has no answer; you must.
- ⚠ **It will happily restart from scratch** rather than extend what you already have. If you are building on existing code, point it there and tell it to extend, not rewrite.
- ⚠ **Stock `Servo.h`** — still wrong, still recommended.

## Done when
- [ ] Your chosen project works and is **demoable**.
- [ ] Input → logic → output loop closed locally.
- [ ] **ThingsBoard shows live telemetry** and a **validated command comes back down**.
- [ ] Every external input validated; out-of-range commands rejected and logged.
- [ ] Induced failures each produce a **documented, safe** outcome.
- [ ] No secrets in source, banner or history; topics namespaced.
- [ ] Architecture page, prompt library and caught-AI log ready to present.

## What a good capstone looks like
| | |
|---|---|
| **Working, defensible device** | demoed — and you can explain every line of it |
| **Prompt library** | templates you reused today *without editing* |
| **"Caught the AI wrong" log** | each story backed by the datasheet line that proves it |
| **Production discipline** | security, error handling, clean FreeRTOS structure |

**The device is the least valuable of the four.** A smaller project with an excellent prompt library,
sharp caught-AI stories and an honest security posture is worth more than a sprawling demo you cannot
explain — and the three that matter are the ones you take back to work.

## Save to your prompt library
- `architecture-review` template · `security-audit` template · `validate-external-input` template — plus
  a final tidy of your whole template library into something you'd hand a colleague on Monday.
