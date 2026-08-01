# AI-Assisted Embedded Development — Framework + ESP32 Course

**Read this before Day 1.** Orientation, the framework, and the hardware reference in one place.

Two things live here, deliberately separated:

- **`framework/`** — a portable, board-agnostic way to drive an AI coding agent (Devin) through an
  embedded build. Reusable on any project, any hardware. Nothing course-specific in it.
- **`course/`** — a specific 3-day ESP32 AI-IoT course delivered *using* that framework.
- **`hardware/`** — the kit facts and spec cards this course runs on. Swap this folder, keep the
  framework, and the method still works.

**No firmware is stored here.** Participants generate all code with the AI. These docs define
*what* to build, *in what order*, and *where the AI will be confidently wrong*.

## Structure
```
.
├── framework/
│   ├── project_starter.json      # copy into every new PlatformIO project — the intake engine
│   └── START_PROMPT.md           # copy-paste prompts: kickoff, build-a-phase, verify, carry-forward
├── course/
│   └── participant/day1-3/       # the nine topic guides (T1-T9)
└── hardware/
    ├── kit_reference_sheet.md    # every pin/address/library/gotcha + failure table
    ├── Robo_ESP32_Rev1.1_Datasheet.md
    └── modules/                  # one spec card per kit part
```

> **Trainer materials — slide decks, BOM and the delivery guide — live on the `trainer` branch**,
> not here. `git checkout trainer` to see them.

---

# 1 · What this course is

You are experienced embedded engineers. This course does **not** teach C, breadboarding, or the OSI
model. It teaches you to **drive an AI coding agent to produce firmware you can defend** — and to
catch it when it is confidently wrong.

> **The thesis, repeated daily:**
> **The AI writes the code; the engineer owns correctness.**

**The shape**
- **9 topics, ~2 hours each, 3 per day.** Each topic is one complete build.
- **Everyone reaches the same objective; everyone's code differs** — because everyone's prompts
  differ. That is expected and it is the point.
- **One new PlatformIO project per topic.** The device grows across the week.

**How you are graded**

| Weight | Criterion |
|---|---|
| 30% | Working, defensible device (demoed) |
| 30% | **Your reusable prompt library** |
| 20% | **"Caught the AI wrong" stories**, verified against a datasheet |
| 20% | Production discipline — security, error handling, clean structure |

**70% of your grade is not the device.** A smaller build with an excellent prompt library and sharp
caught-AI stories beats a sprawling demo you cannot defend. Plan accordingly from Topic 1.

**Two artifacts you build all week — start them on Day 1**
1. **Prompt library** — every prompt that worked, saved as a reusable template.
2. **Caught-the-AI log** — every confident error, *with the datasheet line that proves it* and your
   fix. "The AI was wrong" without proof is an anecdote, not a story.

---

# 2 · What is Devin?

**An autonomous AI software engineer**, not autocomplete. You give it a goal; it plans, writes files,
runs builds and iterates. In this course it runs on the free **SWE-1.6** model.

| | Copilot-style autocomplete | Devin |
|---|---|---|
| Scope | next few lines | whole files, whole projects |
| Context | current buffer | your repo + files you attach |
| Initiative | reactive | plans and executes multi-step work |
| Your job | accept/reject | **specify, review, verify** |

**Genuinely good at:** boilerplate, scaffolding, library glue, refactoring, explaining unfamiliar
APIs, turning a clear spec into working code fast, iterating on a precise error message.

**Where it fails — and this is the course**
- **It confidently invents hardware facts.** Pins, I²C addresses, register names, library names,
  voltages. It does not hedge — it states the wrong thing plainly.
- **It writes happy-path code.** No timeouts, no error branches, no failure states, until you demand them.
- **It defaults to blocking.** `delay()` everywhere, `while(!connected)` loops.
- **It has no idea what is physically wired to your board.** Only you do.

**The mental model:** a fast, knowledgeable, overconfident junior engineer who has never seen your
hardware. Give it a written spec. Review everything against the datasheet.

**One rule you will use nine times:** attaching a file is not an instruction. Devin will *summarize*
an attached JSON unless you tell it to act. Always paste the kickoff prompt.

---

# 3 · What is PlatformIO?

A professional embedded build system inside **VS Code**, replacing the Arduino IDE with a real toolchain.

- **Declared dependencies** — libraries and versions live in `platformio.ini`, in the project, in
  version control. No global Library Manager state that differs between machines.
- **Per-project config** — board, framework, monitor speed, build flags.
- **Real project structure**, working IntelliSense, a usable serial monitor.
- **Reproducible** — a colleague opens your folder and gets your toolchain.

```
my_project/
├── platformio.ini      ← board, framework, lib_deps, monitor_speed
├── src/main.cpp        ← your code
├── include/  lib/  test/
```

```ini
[env:esp32dev]
platform = espressif32
board = esp32dev
framework = arduino
monitor_speed = 115200
lib_deps =
    adafruit/Adafruit NeoPixel
    ; add libraries here as each topic needs them
```

| Action | What it does | Shortcut |
|---|---|---|
| **Build** ✓ | compiles | `Ctrl+Alt+B` |
| **Upload** → | compiles + flashes | `Ctrl+Alt+U` |
| **Monitor** 🔌 | serial at `monitor_speed` | `Ctrl+Alt+S` |
| **Clean** | deletes build artifacts | — |

**Creating a project — you do this 9 times**
1. PlatformIO sidebar → **New Project**
2. Name it (e.g. `T1_gpio`), Board = **Espressif ESP32 Dev Module**, Framework = **Arduino**
3. Set `monitor_speed = 115200`
4. **Upload something trivial and confirm it runs** — *before* the AI is involved

**Gotchas:** the serial monitor holds the port (close it before uploading) · the first build
downloads the toolchain, several minutes, once · if upload fails check you have a **data** cable and
the right COM port.

---

# 4 · The framework — how you actually work

One file, `framework/project_starter.json`, plus copy-paste prompts in `framework/START_PROMPT.md`.
It exists because of one observation:

> **Most AI firmware failures are specification failures.** The AI didn't misunderstand C — it
> guessed a pin, an address, or a voltage that nobody told it.

So we never start with "write me code." We start with an interview.

**Step 0 — You create the PlatformIO project.** Board `esp32dev`, Arduino, 115200. Upload something
trivial and confirm it runs.
> **Why you and not the AI:** a broken build must be unambiguous. If the AI generated the environment
> too, you cannot tell whether a failure is your code, your `platformio.ini`, or your USB driver.
> **You own the environment; the AI owns the code tested against it.**

**Step 1 — Copy `project_starter.json` in, paste the kickoff prompt.** The agent adopts a
senior-engineer role and interviews you: problem, behaviour, hardware, **datasheets**, interfaces,
constraints, failure modes, scope, acceptance.

**Step 2 — Attach the datasheet when asked. This gate is mandatory.** The starter forbids assuming
any pin, address, voltage, direction or timing. Spec cards are in `hardware/modules/`.

**Step 3 — Approve `REQUIREMENTS.md` + `PHASES.md`.** Read them. Correct them. **Nothing is coded
until you approve.** §4 (hard constraints) and §10 (watch-the-AI) are the sections that matter.

**Step 4 — Build one phase at a time.** Never "build everything."

**Step 5 — Verify each phase against the datasheet.** **This is where your graded catches come
from.** Compile ≠ correct.

**Step 6 — Log it.** Prompt-log entry + save any reusable template.

**Repeat runs get faster.** From T2 onward most answers are **"same as \<earlier topic\>"** — attach
that project's `REQUIREMENTS.md` and the AI imports the answer. Target **10–15 minutes**, with depth
going into whatever is *new*. Each topic guide has a *Starter interview — suggested answers* table:
use the shape, in your own words.

---

# 5 · The hardware

### Carrier: Cytron ROBO ESP32 Rev 1.1
Hosts a **30-pin NodeMCU ESP32 DevKit V1**.

| Onboard peripheral | GPIO | Notes |
|---|---|---|
| NeoPixel RGB | D15 | Adafruit NeoPixel |
| Buttons ×2 | **D34, D35** | ⚠ **input-only, no internal pull-up** |
| Piezo buzzer | D23 | has a mute switch — check it before debugging silence |
| Servo headers ×4 | D4, D5, D18, D19 | V+ = supply voltage; the servo plugs straight on |
| I²C | D21 (SDA) / D22 (SCL) | **Grove Port 2 — the only I²C Grove port** |
| Motor driver | D12/D13, D14/D27 | unused in this course |

### Port allocation for the week

| Port | Module |
|---|---|
| **I²C Grove Port 2 (D21/D22)** | **OLED (T3–T7)** → **Gesture (T8–T9)** — one port, swapped once |
| Maker / QWIIC (D21/D22) | *(free — same pins, but a QWIIC connector)* |
| **UART (D16/D17)** | **Grove RS485** |
| Any digital port | Crowtail DHT11 |
| **ADC1 (D32/D33)** | Rotary Angle |
| Servo header | TS90A Micro Servo |

### The six modules

| Module | Interface | Key fact |
|---|---|---|
| **OLED SSD1315** | I²C `0x3C` | SSD1315 works with SSD1306 drivers |
| **Crowtail DHT11** | digital 1-wire | **not I²C**; slow — poll every 1–2 s |
| **Rotary Angle** | analog | **ADC1 only (D32/D33)**; 300°, 10 k |
| **Grove RS485** | UART D16/D17 | 3.3 V/5 V safe; **half-duplex**; A→A, B→B |
| **TS90A Micro Servo** | PWM | **not Grove** — 3-pin lead; 3–6 V; **ESP32Servo**, never `Servo.h` |
| **Gesture PAJ7660** | I²C `0x73` *(verify by scan)* | **5–40 cm** range, plain background |

> **Only one I²C Grove port exists.** The OLED owns it for T3–T7; the gesture sensor takes it from
> T8 and the device runs headless — your T6 dashboard becomes the display. They are never needed at
> the same time, so no I²C hub or QWIIC conversion cable is required.

### ⚠ The seven constraints that break builds

1. **ADC2 dies when WiFi is on.** Analog goes on **ADC1 (D32/D33)**. An ADC2 pin works perfectly on
   Day 1 and returns garbage on Day 2 — it looks exactly like a software bug.
2. **D34/D35 are input-only, no pull-up.** `pinMode(34, INPUT_PULLUP)` is invalid. The AI writes it
   almost every time.
3. **Servo needs `ESP32Servo`.** Stock `Servo.h` will not compile. The TS90A runs from 3–6 V —
   brown=GND, red=V+, orange=signal, and never on the 3V3 Grove rail.
4. **DHT11 is digital 1-wire, not I²C.** Address `0x38` is the **DHT20** — a different part.
5. **RS485 is half-duplex.** Two nodes on one pair; only one may transmit at a time.
6. **The board is 3.3 V.** Grove ports supply 3.3 V and ESP32 inputs are not 5 V tolerant — which is
   why several 5 V-only Grove modules are deliberately not in this kit.
7. **No `delay()` in the main loop.** Non-blocking (`millis()`, later FreeRTOS) only.

Every one is a documented datasheet fact — and every one is something the AI gets wrong.
Full detail, plus a symptom→fix table, in [`hardware/kit_reference_sheet.md`](hardware/kit_reference_sheet.md).

---

# 6 · Before the hands-on

**Assumed — not taught:** C/C++ for embedded (pointers, structs, `volatile`, bitwise) · reading a
datasheet and a pinout · basic electronics · serial debugging as a habit · Git basics (useful, not required).

**Taught as we go:** I²C/UART/SPI differences · interrupts and debouncing · JSON · MQTT · FreeRTOS
tasks and queues.

**Not required:** prior AI-tool experience · fieldbus/RS485 background · prior ThingsBoard use · soldering.

### Setup checklist — before Day 1
- [ ] **VS Code** installed
- [ ] **PlatformIO IDE extension** installed
- [ ] **One project built and uploaded successfully** — downloads the ESP32 toolchain (slow, once)
      and proves your USB drivers work. **Do not skip this.**
- [ ] **Devin access** confirmed, session starts
- [ ] USB **data** cable (not charge-only)
- [ ] Kit checked against the six modules in §5
- [ ] Skimmed [`hardware/kit_reference_sheet.md`](hardware/kit_reference_sheet.md)

### Mindset
- **Compiling is not working.** Working is verified on hardware, against the datasheet.
- **Read `REQUIREMENTS.md` before approving it** — everything downstream inherits its errors.
- **When the AI is confidently wrong, that's a deliverable** — capture it with proof.
- **Ask "what breaks in production?"** about every piece of code it hands you.
- **Descoping is engineering** — it is graded, not penalised.

---

# 7 · The nine topics

Dependency-ordered — nothing is used before it is taught.

| Day | Topic | New knowledge | Hardware |
|---|---|---|---|
| 1 | **T1** GPIO & Digital I/O | prompt skeleton, pins, debounce, interrupt | onboard buttons, buzzer, NeoPixel |
| 1 | **T2** Analog & ADC | ADC1-vs-WiFi, filtering, mapping | Rotary Angle |
| 1 | **T3** Comfort Monitor | I²C bus, DHT 1-wire, non-blocking timers | DHT11, OLED, Rotary |
| 2 | **T4** WiFi | station mode, NVS secrets, backoff reconnect | (+ onboard radio) |
| 2 | **T5** HTTP/REST & JSON | status codes, defensive parse, timeouts | (+ test endpoint) |
| 2 | **T6** MQTT → ThingsBoard | pub/sub, topics, dashboard, RPC | (+ broker/cloud) |
| 3 | **T7** UART & RS485 | half-duplex bus, framing, addressing, fail-safe | Grove RS485 + TS90A servo |
| 3 | **T8** FreeRTOS | tasks, queues, mutex, stack forensics | **+ Gesture PAJ7660** |
| 3 | **T9** Capstone | scoping, integration, security | mini project — no new hardware |

| Day | You end the day with |
|---|---|
| **1** | A local monitor: OLED + DHT11 + knob threshold + alarm |
| **2** | That device live on a cloud dashboard, controllable remotely |
| **3** | Your own scoped mini project — demoed, broken on purpose, and defended |

**Knowledge chain:**
`prompt+pins → digital I/O → analog/ADC → I²C+1-wire → WiFi → HTTP/JSON → MQTT/ThingsBoard → UART/RS485 → FreeRTOS+gesture → scoping/integration/security`

**T9 is a mini project.** Six options (vent controller, cold-chain monitor, access panel, two-node
gateway, calibration rig, or your own) — each needs ≥1 input, ≥1 output, ThingsBoard telemetry and a
validated downlink command. Details in the T9 guide.

---

## Build status
- ✅ All 9 topics — participant guides complete
- ✅ Two slide-deck specs (orientation, FreeRTOS) — on the `trainer` branch
- ✅ `hardware/modules/` — course-written spec cards, one per kit part
- ⏳ MQTT broker + ThingsBoard setup guides — not yet written (the room checklist says what must be
  true, not how to get there)
- ⏳ T5 needs the trainer-hosted test endpoint with four switchable failure modes — the topic is
  hollow without it

## Sources
Hardware spec cards in `hardware/modules/` are original summaries written for this course, each
linking to the manufacturer's page (Seeed Studio, Cytron). `hardware/Robo_ESP32_Rev1.1_Datasheet.md`
is vendor content © Cytron Technologies, used for internal training material.

---

**One last thing.** Nine times this week, the AI will state a hardware fact with total confidence and
be wrong. Every one is catchable by reading the datasheet. That habit — not any single device — is
what you take back to work on Monday.
