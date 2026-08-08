# ESP32 AI-IoT Course — Kit Pin/Library/Address Reference

Carrier: **Cytron ROBO ESP32 Rev 1.1** hosting a 30-pin NodeMCU ESP32 DevKit V1.
Framework: Arduino + PlatformIO (`board = esp32dev`, `monitor_speed = 115200`).

## Onboard peripherals (no purchase)
| Peripheral | GPIO | Notes |
|---|---|---|
| NeoPixel RGB | D15 | Adafruit NeoPixel |
| Buttons x2 | D34, D35 | **INPUT-ONLY, no pull-up** — no `INPUT_PULLUP`, external logic needed |
| Piezo buzzer | D23 | has mute switch |
| Status LEDs | Grove GPIOs | light when GPIO high |
| Dual motor driver | D12/D13, D14/D27 | PWM up to 20 kHz |
| Servo headers x4 | D4, D5, D18, D19 | 3-pin, V+ = supply voltage |
| I2C | D21 (SDA) / D22 (SCL) | Grove Port 2, shared with Maker/QWIIC |

## Purchased modules
| Module | Interface | Address | Port on ROBO | Library | Key gotcha |
|---|---|---|---|---|---|
| OLED SSD1315 | I2C | 0x3C | **Grove Port 2 (D21/D22), T3–T7** | U8g2 **or** Adafruit_SSD1306 + Adafruit_GFX | SSD1315 is SSD1306-compatible; 3.3/5V ok |
| Crowtail DHT11 | digital 1-wire | — | **Grove Port 3 (D26 / D25)** | DHT sensor library | slow (~1 Hz); Grove-compatible connector; poll every 1–2 s |
| RS485 | UART (differential 2-wire bus) | — | UART Grove port (D16/D17) | none — plain `HardwareSerial`; framing is yours | **3.3V/5V supply — ESP32-safe**; **half-duplex**: only one node may drive the bus at a time; **A/B polarity must match** at both ends; both ends must match baud; 120 Ω termination at each end of the trunk |
| Gesture PAJ7660 | I2C (or SPI) | **0x73 — VERIFY by scan** | **Grove Port 2 (D21/D22), T8–T9** | Seeed PAJ7660 lib | 2nd of only two I2C devices; 15+ gestures, **5–40 cm** range |
| Analog Micro Servo 9g (**TS90A**) | PWM | — | onboard servo header (plugs direct — **not Grove**) | **ESP32Servo** | stock `Servo.h` will NOT compile; **3.0–6.0 V — ESP32-safe**; PWM 500–2500 µs = 0–180°; brown=GND, red=V+, orange=signal |
| Rotary Angle | Analog | — | **ADC1 Grove port (D32/D33)** | none (`analogRead`) | **ADC1 ONLY** — ADC2 dies when WiFi is on |

## Critical wiring rules
1. **Only ONE I²C Grove port exists — Port 2 (D21/D22).** The kit has two I²C devices (OLED `0x3C`, Gesture `0x73`), so they cannot both use a Grove port. **They are never needed at the same time:** OLED runs T3–T7, gesture takes Port 2 from T8 onward (the device goes headless and reports to its dashboard). The Maker/QWIIC port shares the same D21/D22 pins but is a **QWIIC/Stemma connector — a Grove module needs a conversion cable to use it**. An I²C Hub or that cable is only required if you want both devices live simultaneously. DHT11 is digital, not on the bus.
2. **Rotary Angle on ADC1 only — D32/D33.** An analog sensor on an **ADC2** pin (D25/D26) works until WiFi is enabled, then reads garbage; it looks exactly like a code bug. Note D25/D26 are ADC2 *and* are taken by the DHT11 here — fine, because the DHT11 is read digitally.
3. **Buttons D34/D35 are input-only, no internal pull-up** — AI will often wrongly emit `pinMode(34, INPUT_PULLUP)`. Teaching moment.
4. **TS90A servo** — a bare 3-pin servo lead, **not a Grove module**: it plugs straight onto a servo header (D4/D5/D18/D19). **Brown=GND, red=V+, orange=signal — the plug is not keyed, check before powering.** Use `ESP32Servo` + `attach(pin, 500, 2500)`. It runs from **3.0–6.0 V**, so Vservo (≈4.6 V on USB) is well inside range — no level shifting needed. Never power it from the 3V3 Grove rail.
5. **Gesture PAJ7660** — I2C on **Grove Port 2 from T8** (the OLED comes off; the device goes headless). Detection range **5–40 cm**; avoid a highly reflective or busy background behind the hand. Address `0x73` is **not definitive in the vendor docs — scan the bus.**
6. **RS485** — UART on D16/D17, **3.3 V/5 V supply so it is ESP32-safe** (unlike the 5 V-only Grove CAN/Bluetooth/RF modules). Two nodes per bus, joined A→A and B→B: **swap the polarity and you get silence, not an error.** It is **half-duplex** — if both nodes transmit at once the bus garbles, so the pair must agree a turnaround scheme. Terminate 120 Ω at each end of a long trunk. Nothing frames your data for you: delimiter, address, length, payload and checksum are yours to design.

## Port allocation (5 Grove ports + servo header)
| Port | Assigned to |
|---|---|
| I2C Grove Port 2 (D21/D22) | **OLED (T3–T7)** → **Gesture PAJ7660 (T8–T9)** — one port, swapped once |
| Maker / QWIIC (D21/D22) | *(free — needs a Grove→QWIIC cable if ever used)* |
| UART (D16/D17) | Grove RS485 |
| **Grove Port 3 (D26/D25)** | Crowtail DHT11 |
| ADC1 (D32/D33) | Rotary Angle |
| Servo header (D4/D5/D18/D19) | TS90A Micro Servo |
| (2 Grove ports spare) | — |

> Prices are estimates — confirm live at cytron.io before ordering. Buy 10–15% spare cables.

## Common failures & fast fixes

| Symptom | Likely cause | Fix |
|---|---|---|
| Analog reads 0/garbage after WiFi on | sensor on ADC2 | move to **ADC1 (D32/D33)** |
| Knob angle wrong but stable | vendor example uses 5 V ref and `/1023` | ESP32 is **3.3 V, 12-bit `/4095`** |
| Button never triggers / always high | `INPUT_PULLUP` on D34/D35 | input-only pins — handle externally |
| Servo won't compile | stock `Servo.h` | use **ESP32Servo** |
| Servo stalls / board resets on move | Vservo sag or inrush on USB power | check supply before blaming code |
| I²C device missing on scan | wrong port, or the other device is fitted | **only Port 2 is I²C**; OLED T3–T7, gesture T8–T9 |
| DHT11 reads NaN / stale | polled too fast, or treated as I²C | poll every 1–2 s; digital pin + DHT library |
| DHT11 reads nothing at all | Seeed library defaults to `DHT 22` | change the definition to **`DHT 11`** |
| Random resets under FreeRTOS | stack too small (**bytes** on ESP32) | raise stack; check high-water mark |
| MQTT drops but the AP is fine | heavy task pinned to Core 0 | Core 0 belongs to the WiFi stack |
| Sparse/stuck dashboard | publish interval or broker auth | non-blocking timer + token in the **username** |
| RS485 pair can't hear each other | baud mismatch or A/B swapped | same baud both ends; **A→A, B→B**; check termination |
