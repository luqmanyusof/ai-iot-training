# Delivering This Course

**Trainer-facing. This branch is not part of `main`.**

### The engineered "catch the AI" moments
The spine of the course — orchestrate them, don't let them pass silently. Each participant guide
carries the full version in its *Catch the AI* section.

| Topic | Trap |
|---|---|
| T1 | `pinMode(34, INPUT_PULLUP)` on input-only pins |
| T2 | Analog on **ADC2**; vendor example using 5 V ref and `/1023` |
| T3 | DHT11 read as I²C / `0x38`; blocking `delay(2000)` |
| T4 | Writes **station mode** for an access-point requirement; polls for a DHCP address on the board that *is* the DHCP server; forgets `handleClient()` |
| T5 | Blocking `while(WiFi.status()...)`; fragile JSON parse — no status check, no `DeserializationError` |
| T6 | ThingsBoard auth — token belongs in the **username**; fails silently |
| T7 | Treats RS485 as plain UART; "add a delay" offered for bus contention |
| T8 | Stack too small (**bytes** on ESP32); gesture address invented; unguarded I²C |
| T9 | Plaintext creds; flat topics; re-adds the OLED that isn't on the bus |

### Room checklist
- [ ] Devin Desktop on **free SWE-1.6** confirmed on delivery day (re-verify — Cognition ships changes).
- [ ] Kits complete: ROBO ESP32 + NodeMCU, Grove cables (10–15% spares), **plus non-Grove items:
      TS90A 3-pin servo leads and twisted pair for the RS485 bus.**
- [ ] Participants have done the §6 setup — **especially the trial upload**.
- [ ] Projector for the shared "it's alive" moments (T6 dashboard, T7 cross-controlled servos).
- [ ] Pre-flight both I²C addresses: OLED `0x3C`, gesture `0x73` (**verify — vendor docs are not definitive**).
- [ ] Broker decision: public (HiveMQ/mosquitto) vs local Mosquitto for air-gapped cohorts.
- [ ] ThingsBoard reachable, and **one device + access token pre-created per participant** — signup
      in class costs 30 minutes and teaches nothing.
- [ ] **T5 hostile endpoint** live with four switchable modes: 500 / HTML / truncated JSON / silent hang.
- [ ] **T7:** node IDs assigned (table in the T7 guide) and **one real RS485 unit checked** for a
      DE/RE direction pin and whether 120 Ω termination is fitted — the vendor pinout is an image only.
- [ ] **T8:** every board can reach the gesture sensor — it takes Grove Port 2 from T8.

### Where to put the emphasis
**There is no grading on this course**, so the only thing steering priorities is what you visibly
value in the room. Be deliberate about it.

- **Spend as much airtime on the prompt library and the caught-AI log as on the demos.** Left alone,
  the room will optimise for a device that lights up — the least portable of the four outcomes.
- **Demand the datasheet line** alongside each caught-AI story — without proof it is an anecdote.
- **Push T9 scope down early.** Call out the first participant who descopes deliberately; that is
  the behaviour you want copied.
- **Ask "which template did you reuse today without editing?"** at every opportunity. A template that
  survived contact with new hardware is the real evidence the method landed.

> ⚠ **Scheduling:** T9's two hours cover the build, **not the presentations**. Budget ~10 min per
> participant separately, or cut the build to 90 minutes. Decide before delivery day.

### Teaching decks
- `trainer/slides/introduction_deck.json` — Day-0 orientation, 26 slides, ~75 min.
- `trainer/slides/t8_freertos_deck.json` — T8 explanation, 22 slides, ~35 min.

---

---

## What else is on this branch

| Path | Purpose |
|---|---|
| `trainer/slides/` | Both deck specs + the images they reference |
| `trainer/BOM.csv` | Kit, SKUs, prices, pin constraints — procurement |
| `trainer/outline_lesson_mapping.md` | Which module teaches which lesson, and why each hardware decision was made |
| `trainer/DELIVERY_GUIDE.md` | This file |

Everything else — the framework, the nine participant guides, and `hardware/` — is shared with
`main`. Keep those edits on `main` and merge forward:

```bash
git checkout trainer
git merge main
```

Because `main` never contains anything under `trainer/`, that merge should never conflict.
