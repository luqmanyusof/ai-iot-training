# Trainer Materials

**This folder exists only on the `trainer` branch.** It is not merged into `main`, so a participant
who clones the default branch never receives it.

| File | What it is |
|---|---|
| [`DELIVERY_GUIDE.md`](DELIVERY_GUIDE.md) | Catch-the-AI table, room checklist, grading notes, scheduling warning |
| [`BOM.csv`](BOM.csv) | Kit, SKUs, prices, pin constraints — procurement |
| [`outline_lesson_mapping.md`](outline_lesson_mapping.md) | Which module teaches which lesson, and the reasoning behind every hardware decision |
| [`slides/`](slides/) | Both deck specs + the images they reference |

## Slide decks

Both are **specs for an AI slide designer**, not rendered decks. They share one `design_direction`
so they look like one system.

| Deck | Slides | Runtime | When |
|---|---|---|---|
| `slides/introduction_deck.json` | 26 + appendix | ~75 min | Day 0, before any hands-on |
| `slides/t8_freertos_deck.json` | 22 + appendix | ~35 min | T8, before the refactor |

`slides/assets/` holds images the decks reference by relative path — Cytron's CDN returns 403 to
non-browser requests, so those two are stored locally rather than hotlinked.

## Working across the two branches

Everything else — `framework/`, `course/participant/`, `hardware/`, `README.md` — is shared with
`main` and identical on both branches.

**Edit shared content on `main`, then merge forward:**

```bash
git checkout main
# ...edit participant guides, framework, hardware...
git commit -am "..."

git checkout trainer
git merge main          # never conflicts: main has nothing under trainer/
```

**Edit trainer-only content on `trainer`** and leave it there. Never merge `trainer` into `main` —
that is what keeps the split working.

## ⚠ A branch is not access control

Anyone who can clone the repository can run `git checkout trainer`. This split keeps trainer material
out of the *default* clone; it does not hide it. If participants must genuinely not see the answer
key, put this content in a **separate private repository** instead.
