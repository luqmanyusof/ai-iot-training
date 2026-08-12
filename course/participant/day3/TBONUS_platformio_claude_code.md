# TBONUS · PlatformIO from the Terminal + Claude Code — A Button-Driven Servo
**Day 3 · bonus · ~90 min (+15 optional) · Participant guide**

## What you'll build
**In plain words:** a button that moves an arm. Press it once and the servo swings from parked to
open; press it again and it swings back. Small on purpose — the build is deliberately the simplest
thing in the course, because the point of this topic is *how* you build it, not *what* you build.

**The moment it works:** you never open the PlatformIO wizard and you never open a browser. You type
three commands in a terminal, run an AI agent inside the folder those commands made, and an arm on
your desk moves when you press a button.

## Objective
The same framework, a different toolchain and a different agent. That is the test: if the method only
works with one IDE and one AI, it isn't a method. By the end you can:
- **Create and drive a PlatformIO project entirely from the command line** — `pio project init`, `pio run`, `pio device monitor` — and read `platformio.ini` as the source of truth it is.
- **Run an agent that lives inside your project** — Claude Code reads your files off disk, edits them in place, and runs your build itself. No attaching, no copy-paste of code.
- **Point that agent at a project that already exists** — your T1–T9 builds — and make it extend them instead of rewriting them.
- **Keep the same gate** — `project_starter.json`, the same interview, the same REQUIREMENTS + PHASES approval before a line of code. Unchanged. That is the deliverable.
- **Name the new hazard** — an agent that can compile will iterate until it compiles. **Compiling is not working, and that gap gets *wider* when the agent can close the compile loop without you.**
- **Move an actuator safely on purpose** — park position on boot, software travel limits inside the mechanical ones, non-blocking motion, and a de-energised resting state.

## Knowledge you'll learn first (in this order)
1. **`pio project init`** — what the CLI actually creates, and why `platformio.ini` is the whole environment in one file you can read, diff and commit.
2. **The build/upload/monitor loop from a terminal** — `pio run`, `pio run -t upload`, `pio device monitor`, and the fact that the monitor holds the COM port.
3. **`lib_deps` from the command line** — `pio pkg search`, and confirming a library's real registry id instead of accepting an invented one.
4. **What changes when the agent is in the room** — filesystem access, in-place edits, and the ability to run commands. What that buys you, and what it costs you.
5. **New project vs existing project** — a green field has nothing to break; a week-old project has working code, real credentials and a git history the agent can also see.
6. **The one boundary that matters here** — **the agent owns the compile; you own the upload.** New code that moves an actuator runs for the first time while you are looking at the actuator.
7. **The servo, properly** — `ESP32Servo` (never `Servo.h`), `attach(pin, 500, 2500)`, and the 3-wire lead that is not a Grove module.
8. **Non-blocking motion** — stepping toward a target on `millis()` instead of a `for` loop full of `delay(15)`.
9. **Safe resting states for something that moves** — boot, fault, and power loss are three different questions with three answers.

## Hardware this topic
| Role | Part | Interface | Where |
|---|---|---|---|
| Input | Onboard button | digital | **D34 — input-only, no pull-up** |
| Output | **TS90A micro servo** (3–6 V, 3-pin lead, **not Grove**) | PWM | servo header — signal on **D4 / D5 / D18 / D19** |
| State | NeoPixel | digital | D15 |
| Console | USB serial | UART0 | 115200 |

> **Nothing else.** No sensor, no network, no display. If your build has more parts in it than this
> table, you have descoped in the wrong direction.
>
> ⚠ **Check the servo lead with the power off before you plug anything in.** **Brown = GND, red = V+,
> orange = signal**, and the connector is **not keyed** — it will go on backwards without resistance.
> V+ on the servo header is the board's supply rail (≈4.6 V on USB), which is inside the TS90A's
> 3.0–6.0 V range. **Never power it from the 3V3 Grove rail.**

## The toolchain — this is the topic

Everything before now started in the PlatformIO wizard and talked to Devin in a browser. This one
does neither.

```bash
mkdir tbonus_servo && cd tbonus_servo

pio project init --board esp32dev \
  --project-option "framework=arduino" \
  --project-option "monitor_speed=115200"

pio run                  # compile — proves the toolchain
pio run -t upload        # flash
pio device monitor       # serial at 115200; Ctrl+C to release the port
```

Then, **in that same directory**:

```bash
cp <course_repo>/framework/project_starter.json .
claude
```

Claude Code now has your project as its working directory. It can read `project_starter.json` without
you attaching it, edit `platformio.ini` and `src/main.cpp` in place, and run `pio run` to see its own
compiler errors. **The kickoff prompt in `framework/START_PROMPT.md` works unchanged** — that is the
proof that the framework is not tied to one agent.

**Add exactly one line to it**, because this agent can act on your disk and Devin could not:

```
Do not run any build, upload or shell command, and do not create or edit any file,
until I have approved REQUIREMENTS.md and PHASES.md.
```

---

## Using Claude Code on a project that already exists

The section above is a green field — nothing to damage. The more common real case, and the one you
will hit on Monday, is pointing an agent at code that already works. **Your T1–T9 projects are
exactly that**, and this is how you open one without regretting it.

```bash
cd T7_rs485            # any existing PlatformIO project
git init && git add -A && git commit -m "before the agent"   # if it isn't already in git
claude
```

**Do these four things before you ask for anything.**

**1 · Get a restore point first.** `git commit` (or a folder copy) before the agent's first edit.
Not because it will definitely break something — because the difference between "the agent changed
something" and "the agent changed *these eleven lines*" is `git diff`, and without a baseline you
don't have it.

**2 · Give it the project's standing facts once, with `/init`.** Run `/init` and Claude Code writes a
`CLAUDE.md` in the project root that it reads automatically on every future session. Edit it down to
what actually constrains this project — it is not a summary, it is a set of rules:

```markdown
# T7 — RS485 cross-control

## Hardware (fixed — never infer these)
- RS485: UART Serial2 on D16/D17. Half-duplex. A→A, B→B. NODE_ID 03.
- Servo: TS90A on servo header D18. ESP32Servo only. attach(pin, 500, 2500).
- Rotary: ADC1 D32 only — ADC2 dies when WiFi is on.
- OLED SSD1315: I2C 0x3C on D21/D22.

## Rules
- No delay() anywhere. Non-blocking motion and non-blocking bus reads.
- Servo parks at 90° on boot and after 3 missed frames.
- Reject frames failing checksum or address — never act on them.
- I run every upload. Do not run `pio run -t upload`.
- Extend the existing code. Do not rewrite files that already work.
```

That block is just **REQUIREMENTS §3 + §4 + §5 compressed**. You already wrote it — you are moving it
somewhere the agent reads without being asked.

**3 · Point it at the specs already on disk, not at your memory.** The carry-forward prompt in
`framework/START_PROMPT.md` §4 works here with the attachments removed, because the files are simply
*there*:

```
Read REQUIREMENTS.md, PHASES.md and CLAUDE.md in this project, and read src/ before changing anything.

I want to <the change>. Do not rewrite working code — extend it.
Tell me which existing functions you will touch and what you will add, and wait for my go-ahead
before you edit. Every hard constraint in REQUIREMENTS §4 and every safety requirement in §5
still applies.
```

**4 · Audit what it can see.** An agent in an existing project directory can read **everything** in
it, and it will — including the ThingsBoard token in your T6 project, a `secrets.h` you meant to
delete, and your git history. You spent a week moving credentials into NVS; this is the moment that
discipline gets tested by something that greps. Check before you `claude`, not after.

**What actually changes, new project vs existing**

| | New project | Existing project |
|---|---|---|
| The gate | full interview → REQUIREMENTS + PHASES | already exist — **read them, then extend** |
| Biggest risk | it invents hardware facts | **it rewrites working code**, or acts on stale assumptions |
| Your restore point | nothing to lose | **`git commit` before the first edit** |
| Standing context | the starter + spec cards | **`CLAUDE.md`** — §3/§4/§5 compressed |
| Secrets exposure | none yet | **everything in the folder and its history** |

> ⚠ **The failure mode here is scale, not error.** In a new project a bad idea produces one bad file.
> In a working project, "clean this up while you're in there" produces a large, plausible diff across
> code you had already verified on hardware — and every line of it compiles. **Ask for one change at
> a time and read the diff**, exactly like a phase.

---

## Your requirement
Build a **two-position servo control**, where:
- The servo **parks at 0° on boot, before the button is read** — the resting state is reached before any input can change it.
- A **debounced press** toggles the target between **PARK (0°)** and **OPEN (90°)**. One press, one move.
- The servo **steps toward its target over time** — visibly smooth, and nothing else in the loop stalls while it moves.
- **Software travel limits are 0–90°**, enforced in code, deliberately narrower than the servo's 180° mechanical range. Any commanded angle outside that range is **rejected and logged**, never clamped into motion.
- Once the servo reaches its target and settles, it is **de-energised** — so an obstructed horn cannot sit drawing stall current.
- The NeoPixel shows state: **parked · moving · open**.
- No `delay()` anywhere, and the button never misses or double-counts.

> **How you capture this:** you have already created the project from the terminal above. Copy
> `framework/project_starter.json` into it, run `claude` in that directory, and paste the kickoff
> prompt from `framework/START_PROMPT.md` plus the extra line. Approve `REQUIREMENTS.md` and
> `PHASES.md` before any code. The interview should be your fastest of the week — everything here is
> either already in a previous project or is on one spec card.

### Starter interview — suggested answers (TBONUS)
| Area | Your answer |
|---|---|
| Problem | A minimal button-driven actuator, built entirely from the terminal with an in-project AI agent, to prove the framework does not depend on a particular IDE or a particular AI. |
| Users *(opt.)* | Bench build; the operator is standing at the device with a hand on the USB cable. |
| Behaviour | Boot → servo parks at 0°, NeoPixel shows parked. Each debounced press flips the target between 0° and 90°; the servo steps toward it on a timer, NeoPixel shows moving, then parked/open. It de-energises once settled. |
| Hardware | ROBO ESP32 + TS90A micro servo on a servo header; onboard button D34 and NeoPixel D15. Nothing else. |
| Documents | `hardware/modules/TS90A-Micro-Servo.md` and `hardware/Robo_ESP32_Rev1.1_Datasheet.md`. **Copy both into the project directory** — this agent reads from disk, it does not take attachments. |
| Interfaces | Button = digital in on **D34, input-only, no internal pull-up**. Servo = PWM on a servo header signal pin (**D4/D5/D18/D19** — state the one you wired), `ESP32Servo`, `attach(pin, 500, 2500)`. NeoPixel D15. |
| Connectivity | None — fully local, no radio at all. |
| Constraints | `ESP32Servo`, never `Servo.h`; non-blocking motion stepped on `millis()`; no `delay()`; `millis()` debounce 20–50 ms; no `INPUT_PULLUP` on D34; servo powered from the servo header rail, never the 3V3 Grove rail; **I run every upload, not the agent**. |
| Safety | **A servo moves a physical arm — this is a real actuator, treat it as one.** **Must never happen:** the servo commanded outside 0–90°, driven into a mechanical end-stop, energised before the system has reached a known state, or left holding torque against an obstruction. Enforce 0–90° in software and reject anything outside it with a serial log — never silently clamp it into motion. **Safe state on boot:** park at 0° before the button is read. **On fault or an out-of-range command:** hold the current target, log, and do not move. **On power loss:** the servo is unpowered and free; on restore it re-parks at 0° before accepting input, so there is no stale commanded position after an unplanned reset. The horn's swing must be physically clear before first motion, and motion must be non-blocking so nothing can delay a stop. |
| Failure modes | Bounce or a held button must never double-toggle. A press during a move is handled by a stated rule — decide it and write it down, do not inherit it by accident. A blocked horn cannot be detected (no position feedback), so bounded travel plus de-energising on settle is the mitigation. A brown-out mid-move resets the board, which re-parks — that is the designed outcome, not a bug. |
| Reuse *(opt.)* | Your `debounce+interrupt` template from T1 and `servo-esp32-nonblocking` from T7 — this project is where you find out whether they survive a different agent unedited. |
| Out of scope | No network, no display, no sensor, no second position source, no FreeRTOS. Full 180° travel is deliberately out of scope. |
| Acceptance | Parks at 0° on boot before any press registers; one press = one move; travel never leaves 0–90°; an out-of-range command is rejected and logged; motion is smooth and nothing blocks; the servo is de-energised at rest; every phase verified by watching the arm. |

## Flow (stages)
- **Stage 0 — Build a project with three commands (15 min):** run the `pio project init` block above, read the `platformio.ini` it generated, then compile and flash a serial banner with `pio run -t upload`. **Test:** the banner appears in `pio device monitor` at 115200. **No servo connected yet, and no AI involved yet** — same rule as every other topic, you own the environment.
- **Stage 1 — Wire the servo with the power off (10 min):** unplug USB. Plug the servo onto one servo header — **brown to GND**, checked twice. Note which signal pin you used. Clear the horn's swing arc. Know how you will cut power in a hurry. Only then plug USB back in. **Test:** the board boots and the banner still prints; the servo may twitch once on power-up and then stay still.
- **Stage 2 — Interview from inside the project (15 min):** copy in `project_starter.json` and the two documents, run `claude`, paste the kickoff prompt plus the do-not-act line. Answer the interview. **Read `REQUIREMENTS.md` §4, §5 and §11 properly before approving** — §5 is the one that matters here, because something moves.
- **Stage 3 — First motion, slow and watched (20 min):** build Phase 1 only. Park at 0° on boot, then **one slow, limited, observed move** — 0° to 30° and back, hands clear. **Test:** the arm moves once, smoothly, and stops where you said. Nothing else. Save a **servo-park-on-boot** template.
- **Stage 4 — The button (20 min):** add the debounced D34 read and the 0°/90° toggle with non-blocking stepping and the NeoPixel state. **Test:** press it twenty times at your natural pace — twenty moves, no missed presses, no doubles, no stalling. Then press it fast, mid-move, and confirm the rule you wrote down is the rule it follows.
- **Stage 5 — Break it on purpose (10 min):** press repeatedly during a sweep · power-cycle mid-move and watch it re-park at 0° · ask the agent for a `moveTo(150)` and confirm it is **rejected and logged, not clamped** · then **briefly** obstruct the horn with a finger, confirm it de-energises on settle rather than pushing, and let go. Keep a hand near the USB plug for that last one.
- **Stage 6 — Open an old project (15 min, optional):** `cd` into any T1–T9 project, commit it, run `/init`, trim the generated `CLAUDE.md` to the rules above, and ask for **one small extension** — a serial command, an extra NeoPixel state. **Test:** it builds, it still passes that project's original acceptance criteria on hardware, and `git diff` shows only the lines you agreed to. **The diff is the test here, as much as the device.**

## Catch the AI
This topic has two sets of traps: the servo ones, which are the same as always, and a new set that
only exists because the agent is inside your project.

**New, and specific to an agent that can run your build**
- ⚠ **The headline trap: it fixes the build error, not the design error.** Claude Code can run `pio run`, read the failure and try again — so it will keep iterating until the code compiles. That is genuinely useful, and it is also how you end up with immaculate, warning-free code driving the wrong pin at the wrong angle. **A green build is now the agent's success criterion; it must never be yours.** Every phase still ends with you watching the arm.
- ⚠ **It uploads to your board.** Left unrestricted it will run `pio run -t upload` to "check its work" — flashing new actuator code while you are looking at your screen instead of the servo. **Hold the upload yourself.** The agent owns the compile; you own the flash and the eyes on the hardware.
- ⚠ **It acts during the interview.** Devin could only talk. This one can write `main.cpp` while it is still asking you questions. That is what the extra kickoff line is for — and check that it obeyed it before you approve anything.
- ⚠ **It re-runs `pio project init`** or rewrites `platformio.ini` from memory, quietly changing the board id or dropping `monitor_speed`. You created that file. Diff it.
- ⚠ **In an existing project, it rewrites rather than extends.** Ask for one addition and get a refactor of three files that already worked and were already verified on hardware. Everything compiles, so nothing warns you. **Ask for one change, read the diff, and say "extend, do not rewrite" in the prompt.**
- ⚠ **It trusts a stale comment over the datasheet.** In a week-old project it reads your old code and your old comments as fact — including anything that was wrong and never bit you. `CLAUDE.md` and REQUIREMENTS §3 are the authority; say so.
- ⚠ **Invented library ids.** It will write a plausible `lib_deps` line with a plausible version pin. Confirm with `pio pkg search esp32servo` and use what the registry actually returns.
- ⚠ **The serial monitor holds the port**, so the agent's upload fails with a permissions error and it will confidently diagnose a driver problem, a cable problem, or a board problem. It is none of those. Close the monitor.

**The servo traps — unchanged, and it still falls for them**
- ⚠ **Stock `Servo.h`.** It does not compile on ESP32 and the AI still recommends it first. Use **ESP32Servo**.
- ⚠ **`attach(pin)` with no pulse range.** The TS90A is **500–2500 µs**; the library default gives you a fraction of the travel you asked for, and it looks like a mechanical fault.
- ⚠ **`pinMode(34, INPUT_PULLUP)`** — third time this week, and here it **compiles cleanly and is silently wrong**, which is exactly the point of this topic.
- ⚠ **A blocking sweep** — `for (int a = 0; a <= 90; a++) { servo.write(a); delay(15); }` stalls the whole device for over a second per move. Demand stepping on `millis()`.
- ⚠ **No park on boot.** The servo does whatever the last power-up left it doing, which for an actuator is an undefined state, not a cosmetic one.
- ⚠ **Silently clamping an out-of-range angle** into a legal one instead of rejecting and logging it. It will call that "safe". It hides the bug that sent 150° in the first place.
- ⚠ **It treats the servo as a Grove module** or powers it from 3V3. It is a bare 3-pin lead on the servo header, and 3V3 is limited to 300 mA across all Grove ports.
- ⚠ **Unexplained resets when the servo starts** — it will blame your code. Suspect **inrush current browning out the board on USB power** first.

## Done when (shared objective)
- [ ] The project was created with **`pio project init` from a terminal**, and you can explain every line of the `platformio.ini` it produced.
- [ ] `project_starter.json` ran the **same interview** it runs for Devin, and `REQUIREMENTS.md` + `PHASES.md` were approved **before any code existed**.
- [ ] Servo **parks at 0° on boot**, before the button is read.
- [ ] One press = **one move**, PARK ⇄ OPEN, no missed presses and no doubles.
- [ ] Motion is **non-blocking and visibly smooth**; no `delay()` anywhere.
- [ ] Travel never leaves **0–90°**; an out-of-range command is **rejected and logged**, not clamped.
- [ ] The servo is **de-energised once settled**, and a brief obstruction does not leave it straining.
- [ ] A power cycle mid-move ends with the arm **re-parked at 0°**.
- [ ] **Every upload was run by you**, with your eyes on the arm.
- [ ] You can state at least one thing the agent got **compiling and wrong** — with the spec-card line that proves it.
- [ ] *(optional)* An existing project opened with a **restore point and a `CLAUDE.md`**, extended by one change, and still passing its original acceptance criteria on hardware.

## Save to your prompt library
- `pio-cli-init` template (the three commands + the `platformio.ini` you want) · `claude-md-project-rules` template (§3/§4/§5 compressed into standing agent context) · `extend-dont-rewrite` template · `servo-park-on-boot` template · `servo-nonblocking-step` template · `reject-dont-clamp` template — the last one applies to every actuator command you will ever validate.

**What this topic is really for.** You now have the same build gate running against two different
agents in two different toolchains, on a new project and on an old one. If your prompt library
survived the swap unedited, it is a library. If it only worked in one browser tab, it was a
transcript.
