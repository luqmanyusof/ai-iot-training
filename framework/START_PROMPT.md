# START_PROMPT — copy-paste prompts for driving the AI

Attaching `project_starter.json` on its own does **not** work. The agent treats an attached JSON as
a document to read and summarize; nothing in a data file tells it to *act*. You have to give it the
instruction. These are the four prompts for a whole project, in order.

---

## 0. Project setup — do this BEFORE any prompt

**Create the PlatformIO project yourself. Do not ask the AI to scaffold it.**

1. **PlatformIO → New Project** — board `esp32dev`, framework `Arduino`. Let PIO generate
   `platformio.ini`, `src/`, `include/`, `lib/` and the VS Code config.
2. Set `monitor_speed = 115200` in `platformio.ini`.
3. **Prove the toolchain before the AI is involved** — build and upload anything trivial (serial
   banner, NeoPixel blink). If this does not flash, stop and fix it now.
4. **Copy `project_starter.json` into the project root.**
5. Only now paste the kickoff prompt below.

**Why this order:**
- A broken build must be unambiguous. If the AI generated the environment too, you cannot tell
  whether a failure is your code, your `platformio.ini`, or your USB driver.
- `platformio.ini` is the one file the AI should never invent — wrong `board` id, missing
  `framework`, or hallucinated `lib_deps` versions all fail confusingly at build time.
- PIO's wizard creates `include/`, `lib/`, `test/` and `.vscode/`; an AI usually writes only
  `platformio.ini` + `main.cpp`, which degrades IntelliSense and makes you debug the IDE.
- Clean ownership: **you own the environment, the AI owns the code tested against it.** The AI still
  *edits* `platformio.ini` in Phase 1 to add `lib_deps` — it just does not create the project.

---

## 1. Kickoff — starts the interview

> Copy everything in the box. Make sure `project_starter.json` is in the project (or attached).

```
Read project_starter.json. It is your OPERATING INSTRUCTIONS for this session, not a document to summarize.

Act as a SENIOR EMBEDDED SYSTEMS ENGINEER scoping this build with me.
Do NOT write any code. Do NOT create REQUIREMENTS.md or PHASES.md yet.
Do NOT summarize the starter back to me.

Start now:
1. Reply with ONE line confirming the starter is loaded.
2. Then ask me interview question 1 (area "problem") — that question only.
3. Stop and wait for my answer. Ask exactly one question per message for the whole interview.

For the rest of this session you must follow the starter's `role_instruction` and `interview_rules`:
- One question at a time; a one-line answer like the `hint` is enough.
- Reflect each answer back in one line so I can correct you.
- Optional questions may be skipped if they clearly do not apply — say which you are skipping.
- If I say "same as <project>" and attach a REQUIREMENTS.md, import that answer instead of re-asking.
- NEVER assume a pin, address, voltage, direction or timing. Ask me for the datasheet first, and
  extract everything in `must_extract_from_docs` from it.

When the interview is done: summarize the full scope in one block, wait for my approval, and only
then write REQUIREMENTS.md and PHASES.md per `output_spec` into the project root. Then stop.
```

**If it still summarizes instead of asking**, reply with:

```
Stop. Do not describe the file. Ask me interview question 1 now, and nothing else.
```

---

## 2. Build a phase — use once per phase, never "build everything"

```
Read REQUIREMENTS.md and PHASES.md in this project.

Build ONLY Phase <N>. Do not start any later phase, and do not refactor earlier phases.

Rules:
- Every hard constraint in REQUIREMENTS §4 applies to this code. Re-read them before you write.
- Respect the engineering principles: non-blocking, tiny ISRs, secrets out of source, all external
  input untrusted until validated.
- Use the exact pins, addresses and libraries in REQUIREMENTS §3 — if something is not stated there,
  ask me instead of choosing for me.

When you are done, tell me:
1. What you built, in three lines.
2. Which REQUIREMENTS §4 constraints your code satisfies, and how.
3. Anything you had to assume — flag it explicitly so I can check it against the datasheet.
```

---

## 3. Verify a phase — the "catch the AI" step

> Run this **before** you tick the phase off. This is where the graded catches come from.

```
Review the code you just wrote for Phase <N> against REQUIREMENTS.md and the datasheets I attached.

Check specifically:
- Does every pin you used match the datasheet, including DIRECTION and pull-up availability?
- Does every address / interface / link parameter match the document, not your memory?
- Any blocking call (delay, while-wait) that violates §4?
- Any external input used without validation?
- Anything in REQUIREMENTS §10 "watch the AI" that you have just done anyway?

For each issue: quote the line, quote the datasheet fact that contradicts it, and propose the fix.
If you find nothing, say so explicitly — do not invent issues.
```

Then log the result in the PHASES.md prompt-log table — including the times **you** caught it and it
did not catch itself.

---

## 4. Carry forward to the next project

> Use this as the answer to the `reuse` interview question in your next build.

```
Here is my REQUIREMENTS.md from <previous project> and my prompt template library.
For this build, treat as "same as before": <list the areas — e.g. hardware, interfaces, constraints>.
Import those answers instead of re-interviewing me, and only ask me about what is NEW this time.
```

---

## Why this exists

The starter carries an `AI_READ_THIS_FIRST` block that tells a compliant agent to activate itself,
but agent behaviour with attached files varies — some read attachments lazily, some summarize by
default, some only read a file when a prompt refers to it. **The kickoff prompt removes the
ambiguity.** Treat the JSON as the specification and this file as the ignition.
