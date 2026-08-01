# T8 · FreeRTOS — Concurrency Without Mystery Reboots
**Day 3 · ~2 hours · Participant guide**

## Objective
Refactor a week of single-loop code into production structure. By the end you can:
- **Create tasks deliberately** — `xTaskCreatePinnedToCore` with a justified stack size (bytes on ESP32), priority and core.
- **Move data with queues** — not shared globals; know who may block and what happens when a queue fills.
- **Protect shared hardware** — a mutex on the I²C bus, having seen the corruption without it.
- **Diagnose stack overflow** — the failure that masquerades as a faulty board; `uxTaskGetStackHighWaterMark` is the instrument.
- **Respect the cores** — WiFi lives on Core 0; starve it and your "network problem" is actually scheduling.
- **Prove the payoff** — add a brand-new sensor to a well-structured system and watch how cheap it is.

## Knowledge you'll learn first (in this order)
1. **You've been using FreeRTOS all week** — `loop()` runs inside a task the Arduino core created for you.
2. **Tasks** — `xTaskCreatePinnedToCore()`, priorities, and the **stack size in words vs bytes** confusion that bites everyone.
3. **The two cores** — Core 0 runs the WiFi/BT stack; Core 1 runs your `loop()`. Pinning matters, and starving Core 0 breaks the radio.
4. **`vTaskDelay()` vs `delay()`** — one yields the CPU to other tasks, one is now the right tool. Your "no `delay()`" rule changes shape here.
5. **Queues** — the correct way to move sensor data between tasks. Shared globals are a race, not a design.
6. **Mutexes** — the I²C bus is a shared resource; two tasks touching `Wire` concurrently corrupt each other's transactions.
7. **Event-driven input** — a gesture task that blocks waiting for something to happen, instead of polling in the main loop.
8. **Stack overflow diagnosis** — `uxTaskGetStackHighWaterMark()`, and why it looks like random resets, boot loops, or "the board is faulty."
9. **Watchdog** — why a tight loop with no yield triggers the task watchdog timer.

## Hardware this topic
| Role | Part | Where |
|---|---|---|
| Carrier + WiFi | ROBO ESP32 | — |
| **Gesture (NEW this topic)** | **Grove Smart IR Gesture PAJ7660** | I²C **`0x73` — verify by scan**, Grove Port 2 (D21/D22) |
| ~~Display~~ | ~~OLED~~ — **comes off now**; the gesture sensor takes the single I²C port. Your T6 dashboard is the display. |
| Environment | Crowtail DHT11 | digital 1-wire |
| Threshold knob | Rotary Angle | ADC1 (D32/D33) |
| Alarm | Buzzer + NeoPixel | D23 / D15 |
| Optional | RS485 bus + servo (from T7) | UART D16/D17 · servo header |

## Your requirement
Rebuild your T6 monitor so that:
- It runs as **five or more tasks** — sensor, gesture, network/publish, alarm/control, diagnostics.
- Data crosses tasks via **queues**; no unprotected shared globals.
- The I²C bus is **mutex-protected** — you will *create* that contention yourself when the gesture task lands.
- A **new input works**: a swipe silences the alarm (or steps the threshold) — added *after* the task structure exists, to show how cheap that is.
- The device is **headless** — local state on NeoPixel + buzzer, full state on your T6 dashboard.
- Every task reports its **stack high-water mark**; sizes are justified by measurement.
- It survives a **15-minute soak** with WiFi/MQTT active — zero resets, zero watchdog trips.

> **How you capture this:** create a **new PlatformIO project** (`esp32dev`, and prove the toolchain uploads — see `framework/START_PROMPT.md` §0), copy `framework/project_starter.json`
> into it, and paste the kickoff prompt from `framework/START_PROMPT.md` into Devin. This is a refactor — most answers are "same as T6" (attach that
> project's `REQUIREMENTS.md`); the new material is the task/queue/stack constraints, and the board
> datasheet in `hardware/` covers the dual-core layout. Approve both files before any code.

### Starter interview — suggested answers (T8)
| Area | Your answer |
|---|---|
| Problem | The T6 monitor refactored into FreeRTOS tasks, then extended with a gesture input to prove the structure pays off. |
| Users *(opt.)* | Same device; it is heading into a product, so structure now matters. |
| Behaviour | T6 behaviour, headless, **plus** a swipe that silences the alarm / steps the threshold. Internally: sensor task → queue → network + control tasks; gesture task event-driven; alarm task highest relevant priority; diagnostics print HWM + heap. |
| Hardware | Same as T6, **minus the OLED, plus the Gesture PAJ7660** on the single I²C port. |
| Documents | T6 REQUIREMENTS.md + the board datasheet (dual-core layout) + `hardware/modules/Grove-Gesture_sensor_paj7660.md`. |
| Interfaces | Gesture on I²C **`0x73` (verify by scan)**, Grove Port 2; no OLED; I²C is now a resource shared between tasks. |
| Connectivity | Same as T6. |
| Constraints | Stacks sized from `uxTaskGetStackHighWaterMark` (argument is **bytes** on ESP32); queues not globals; mutex on `Wire`; `vTaskDelay` only; heavy work off Core 0; `loop()` essentially empty. |
| Failure modes | Any reset must be explainable from the panic backtrace; queue-full policy defined; zero watchdog trips in the soak. |
| Reuse *(opt.)* | T6 REQUIREMENTS wholesale — this is a refactor, not a feature build. |
| Out of scope | One new input only — no OTA, no power management, no new cloud features. |
| Acceptance | ≥5 justified tasks; queue data flow; mutex proven (race seen without it); gesture working and debounced; HWM documented; deliberate overflow caused + read; 15-min clean soak. |

## Flow (stages)
- **Stage 0 — Baseline & map (10 min):** flash your working T6 device. On paper, split it into tasks: what runs at what rate, what data crosses between them, what shares hardware.
- **Stage 1 — Your first task (20 min):** prompt the AI to move the sensor read into its own task with `xTaskCreatePinnedToCore`. Ask it to justify the stack size and the core it chose — do not accept "1024" without a reason. Save a **freertos-task** template.
- **Stage 2 — Split it up (25 min):** move network, alarm/control and diagnostics into their own tasks at sensible priorities. `loop()` should end up nearly empty.
- **Stage 3 — Queues, not globals (25 min):** replace shared variables with a queue carrying a reading struct. Discuss what happens when the queue fills and who is allowed to block.
- **Stage 4 — New hardware, and the race it creates (25 min):** swap the OLED off Port 2 and plug in the **gesture sensor**. Scan the bus, confirm **`0x73`**, then add a **gesture task** that waits for input and posts events to a queue — a swipe silences the alarm. Notice how little of your existing code changes: *that* is the payoff of Stage 2. Now two tasks touch `Wire`, so **guard it with a mutex** — then remove the mutex deliberately and watch reads garble and gestures go missing. That race is real, and you just built it.
- **Stage 5 — Stack forensics (15 min):** print every task's high-water mark. Then **shrink one task's stack until it crashes** and read the panic output. Size all stacks from measurement, not guesswork, and soak-test.

## Catch the AI
- ⚠ **The headline trap: stack too small.** The AI hands out `1024` as a default stack size. Add a `Serial.printf`, an ArduinoJson document, or an OLED render inside that task and it overflows — and the symptom is a **boot loop or random resets that look exactly like a faulty board or bad power supply**. Engineers lose hours here blaming hardware. Print the high-water mark, size from evidence, log the catch.
- ⚠ **Stack units.** The parameter is in **words on some ports, bytes on ESP32** — the AI mixes these up and its "safe" number is off by 4×.
- ⚠ **Shared globals with no protection**, often `int` flags with no `volatile` and no queue. It will tell you it's fine. It isn't.
- ⚠ **Unguarded I²C from two tasks** — garbled reads and dropped gestures, and the AI will suggest you "add a delay."
- ⚠ **Gesture address invented.** It will state an I²C address with total confidence. **Scan for `0x73`** — the datasheet itself marks it "verify."
- ⚠ **Gesture polled in a hot loop** instead of an event-driven task that yields — a classic way to trip the task watchdog.
- ⚠ **Wrong core pinning** — heavy work pinned to Core 0 starves the WiFi stack; your MQTT drops and looks like a network fault.
- ⚠ **`delay()` still in tasks**, or a task with no yield at all → task watchdog reset.
- ⚠ **Priority inversion by accident** — everything created at the same priority, or the alarm task lowest.

## Done when (shared objective)
- [ ] Five or more tasks, each with a **justified** stack size and core assignment.
- [ ] Sensor data crosses tasks via a **queue**, not a shared global.
- [ ] Gesture confirmed at **`0x73`** by scan and driving a real action (swipe silences the alarm).
- [ ] I²C access protected by a **mutex** (and you have seen the race without it).
- [ ] Every task prints its **stack high-water mark**; headroom is documented.
- [ ] You have **deliberately caused and read** a stack-overflow panic.
- [ ] `loop()` is essentially empty; no `delay()` inside tasks.
- [ ] 15-minute soak with WiFi/MQTT up: no resets, no watchdog.
- [ ] You can state **how few files changed** to add the gesture — the payoff of the structure.

## Save to your prompt library
- `freertos-task` template · `queue-between-tasks` template · `mutex-shared-bus` template · `event-driven-input-task` template · `stack-sizing-checklist` (the high-water-mark habit is the one you'll reuse most).
