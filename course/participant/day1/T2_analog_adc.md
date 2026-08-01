# T2 · Analog Sensing & ADC
**Day 1 · ~2 hours · Participant guide**

## Objective
Same objective for all; different code is fine. By the end you can:
- **Read analog correctly on the ESP32** — 12-bit (0–4095), attenuation, and the non-linearity near the rails.
- **Choose the right ADC** — ADC1 vs ADC2, and why an ADC2 pin silently dies the day WiFi turns on.
- **Filter a noisy signal** with a moving average until the plotter trace is stable.
- **Map and calibrate** raw counts into a meaningful 0–100 value that drives an output smoothly.
- **Bank two prompt templates** — analog-read and moving-average-filter — for every analog sensor you'll ever prompt for.

## Knowledge you'll learn first (in this order)
1. **The ESP32 ADC** — 12-bit (0–4095), input attenuation, and why it's **non-linear**.
2. **ADC1 vs ADC2** — and the trap: **ADC2 stops working once Wi-Fi is on** (you'll rely on this from Day 2).
3. **Moving-average filtering** — turning a jittery reading into a stable one.
4. **Mapping / calibration** — raw counts → a meaningful 0–100 range.

## Hardware this topic
| Role | Part | Pin |
|---|---|---|
| Analog input | Rotary Angle | **ADC1 Grove port (D32/D33)** |
| Visual output | NeoPixel RGB | D15 |
| Debug | Serial plotter | USB |

## Your requirement
Build a device where:
- The knob position reads as a **smoothed 0–100 value** on the serial plotter.
- The same value drives **NeoPixel brightness or hue** live as you turn.
- The reading is **stable** — no visible jitter after filtering.
- The knob is wired and read on an **ADC1 pin (D32/D33)** — non-negotiable.

> **How you capture this:** new PlatformIO project (created by you, toolchain proven — `framework/START_PROMPT.md` §0), copy `framework/project_starter.json` in, kick off with `framework/START_PROMPT.md`, run the
> interview ("same as T1" covers the board). Approve `REQUIREMENTS.md` + `PHASES.md` before coding.

### Starter interview — suggested answers (T2)
| Area | Your answer |
|---|---|
| Problem | A knob that reads as a stable 0–100 value and drives the NeoPixel — the analog input layer for later topics. |
| Users *(opt.)* | Same bench context as T1 — skip. |
| Behaviour | Turn knob → raw ADC filtered (moving average, window ~10) → mapped 0–100 → plotter + NeoPixel follow smoothly. |
| Hardware | ROBO ESP32 + Grove Rotary Angle; onboard NeoPixel (D15). |
| Documents | Attach `hardware/modules/Grove-Rotary_Angle_Sensor.md`; board datasheet same as T1. |
| Interfaces | Rotary = analog on **ADC1 (D32/D33) only**; NeoPixel = D15. |
| Connectivity | None. |
| Constraints | ADC1 only — ADC2 dies when WiFi turns on (Day 2 depends on this); 12-bit 0–4095; non-blocking loop. |
| Failure modes | Raw jitter must never reach the output — filter first; note the flat zones near 0/4095. |
| Reuse *(opt.)* | T1 REQUIREMENTS (board facts) + digital-output template. |
| Out of scope | No network, no display, no thresholds (T3 adds those). |
| Acceptance | Reading on ADC1; smoothed trace visibly cleaner than raw; stable 0–100; NeoPixel follows without steps or flicker. |

## Flow (stages)
- **Stage 1 — Raw read (25 min):** prompt for `analogRead` on the knob, stream to the **serial plotter**. Observe the jitter. Save an **analog-read** prompt template.
- **Stage 2 — Filter (30 min):** prompt to add a **moving-average** filter (window N, default 10). Plot raw vs smoothed on the same graph.
- **Stage 3 — Map & calibrate (25 min):** map the smoothed value to **0–100**. Discuss the ESP32 ADC non-linearity near the rails.
- **Stage 4 — Drive output (25 min):** use the 0–100 value to set NeoPixel **brightness or hue** live as you turn the knob.

## Catch the AI
- ⚠ Check which pin the AI chose. If it picked an **ADC2** pin (D25/D26), it "works" today but will read **garbage the moment Wi-Fi turns on** in Day 2. Force it onto **ADC1 (D32/D33)** now. Log it.

## Done when (shared objective)
- [ ] Knob is read on an **ADC1** pin.
- [ ] Serial plotter shows a **smoothed** trace with far less jitter than raw.
- [ ] Value maps to a stable **0–100**.
- [ ] NeoPixel responds smoothly as you turn the knob.

## Save to your prompt library
- `analog-read` template · `moving-average-filter` template.
