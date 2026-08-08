# T1 · Prompt-Driven GPIO & Digital I/O
**Day 1 · ~2 hours · Participant guide**

## What you'll build
**In plain words:** a small arming panel. Press the button once — it beeps, and the light changes
colour. Press again and it moves to the next colour, cycling through three states the way an alarm
panel goes from off, to armed, to alarming.

**The moment it works:** press the button as fast as you can. One press, one beep, every single
time — no missed presses, no accidental doubles.

## Objective
Everyone reaches the same objective; your code will differ because your prompts differ — that's expected. By the end you can:
- **Prompt with the skeleton** — platform + framework + exact pins + behaviour + constraints + library + output format — instead of vague asks.
- **Produce debounced, interrupt-driven input** that never misses or double-counts a press (`IRAM_ATTR` ISR + `volatile` flag, work done in `loop()`).
- **Catch the AI on the board's pins** — D34/D35 are input-only with no pull-up; `INPUT_PULLUP` there is confidently wrong.
- **Run a 3-state machine** (IDLE → ARMED → ALARM) with zero `delay()` in the loop.
- **Start your prompt library** — the deliverable that outlasts the device and comes to work with you.

## Knowledge you'll learn first (in this order)
1. **The AI prompt skeleton** — PLATFORM + FRAMEWORK + EXACT PINS + BEHAVIOUR + CONSTRAINTS + LIBRARY + OUTPUT FORMAT.
2. **The ROBO ESP32 pin map** and which pins are special.
3. **Input-only pins** — D34/D35 have no internal pull-up.
4. **Debounce** — why a mechanical button "bounces" and how `millis()` fixes it.
5. **Hardware interrupts** — `attachInterrupt`, `IRAM_ATTR`, `volatile`, keep the ISR tiny.

## Hardware this topic
| Role | Part | Pin |
|---|---|---|
| Input | Onboard buttons | D34 / D35 (**input-only**) |
| Audible output | Piezo buzzer | D23 |
| Visual output | NeoPixel RGB | D15 |

## Your requirement
Build a device where:
- A **clean button press** produces exactly **one beep** — no missed presses, no doubles.
- Each press advances a 3-state indicator **IDLE → ARMED → ALARM → IDLE**, shown as NeoPixel colour.
- Input is **interrupt-driven and debounced** (~20–50 ms window).
- ALARM has a distinct buzzer tone/pattern.
- Nothing blocks — no `delay()` in the main loop.

> **How you capture this:** create a **new PlatformIO project** (`esp32dev`, and prove the toolchain uploads — see `framework/START_PROMPT.md` §0), copy `framework/project_starter.json`
> into it, and paste the kickoff prompt from `framework/START_PROMPT.md` into Devin. It interviews you like an engineer and generates `REQUIREMENTS.md` +
> `PHASES.md` in the project root; you approve both before any code is written. Use the table below
> to answer — in your own words.

### Starter interview — suggested answers (T1)
| Area | Your answer |
|---|---|
| Problem | A button-driven 3-state arming panel: each clean press beeps once and advances IDLE → ARMED → ALARM. |
| Users *(opt.)* | Facility operator; indoor bench build. |
| Behaviour | Press → one beep + next state; NeoPixel colour per state (off/green/red); ALARM adds a buzzer pattern. |
| Hardware | ROBO ESP32 only — onboard button (D34/D35), buzzer (D23), NeoPixel (D15). |
| Documents | Attach `hardware/Robo_ESP32_Rev1.1_Datasheet.md`. |
| Interfaces | Button = digital in **D34, input-only, no pull-up**; buzzer = digital out D23; NeoPixel = D15 (Adafruit NeoPixel). |
| Connectivity | None — fully local. |
| Constraints | No `delay()` in loop; `millis()` debounce 20–50 ms; ISR = `IRAM_ATTR`, sets a `volatile` flag only; no `INPUT_PULLUP` on D34/D35. |
| Failure modes | Bounce or electrical noise must never double-count; no missed presses at 5 presses/s. |
| Reuse *(opt.)* | Nothing yet — this project starts the prompt library. |
| Out of scope | No sensors, no network, no display. |
| Acceptance | 1 press = 1 beep across 100 presses; 3-state cycle correct; interrupt + `volatile` flag verified; zero `delay()` in loop. |

## Flow (stages)
- **Stage 0 — Prereq check (10 min):** flash the onboard NeoPixel colour cycle to prove the toolchain. Do not linger.
- **Stage 1 — Prompt skeleton (25 min):** learn the skeleton; prompt the AI for a NeoPixel red/green/blue cycle. Save this as your **digital-output** prompt template.
- **Stage 2 — Read a button (20 min):** prompt for a polled button read on D34 that beeps the buzzer. Press it fast — observe missed/double beeps (bounce).
- **Stage 3 — Debounce (25 min):** prompt to add a `millis()` debounce. Now one clean press = one beep.
- **Stage 4 — Interrupt (25 min):** convert to an interrupt: `IRAM_ATTR` ISR sets a `volatile` flag, handle it in `loop()`. Discuss why the ISR must stay tiny.
- **Stage 5 — State machine (15 min):** extend to IDLE → ARMED → ALARM, one state per press, NeoPixel colour per state, buzzer on ALARM.

## Catch the AI
- ⚠ The AI will likely emit `pinMode(34, INPUT_PULLUP)`. **D34/D35 are input-only with no pull-up** — that's wrong. Ask it how it handles the missing pull-up, and fix it. Log this in your prompt log.

## Done when (shared objective)
- [ ] One clean press = exactly one beep (no bounce).
- [ ] Input is interrupt-driven (`IRAM_ATTR` ISR + `volatile` flag).
- [ ] Press cycles IDLE → ARMED → ALARM with distinct NeoPixel colours.
- [ ] No `delay()` in the main loop.

## Save to your prompt library
- `digital-output` template · `debounce+interrupt` template.
