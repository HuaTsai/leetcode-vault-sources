---
name: leetcode-tutor
description: >
  Spaced-repetition review tutor for this LeetCode Obsidian vault. Use when the user wants to:
  (1) Review problems that are due / about to be forgotten,
  (2) Drill a weak algorithm pattern,
  (3) Blind-solve a problem from memory and get graded,
  (4) Take a quick MCQ check (pattern recognition, complexity, key insight, bug spotting),
  (5) Pick up new problems from the curriculum,
  or says things like "quiz me", "test me", "複習", "出題", "考我", "/leetcode-tutor".
---

# LeetCode Tutor Skill

Spaced-repetition (SR) tutor for a LeetCode problem vault. It tracks **per-problem recall
strength** and surfaces problems that are about to be forgotten, then drills them.

## Core design — read this first

This vault has an **essential difference** from a concept-knowledge vault:

1. **Procedural, not declarative.** The skill being tested is "can you _solve / recall the
   approach / implement_ it", not "do you understand a concept". So the primary mode is
   **blind-solve** (free response, graded against the stored solution), with MCQ used only as a
   fast supplement.
2. **State lives OUTSIDE note frontmatter.** Problem notes (`<Pattern>/NNNN-*.md`) are durable,
   published artifacts (`dg-publish: true`). Review state changes every session, so keeping it in
   frontmatter would churn publishable files on every review. Therefore **ALL SR state lives in
   `Progress/` (gitignored)**, NOT in note frontmatter.
   - **NEVER write `solved-*`, `box`, `next-due`, etc. into a problem note's frontmatter.**
   - Note frontmatter keeps only intrinsic fields: `leetcode-id`, `difficulty`, `tags`, `memo`,
     `dg-publish` (and optionally a write-once `first-solved`).

## File structure

```
<cwd>/                          ← the vault root
├── <Pattern>/NNNN-Title.md     ← problem notes (Hashing/, Heap/, Sliding Window/, ...)
├── problemsets.toml            ← curriculum (Grind 169 / NeetCode 150)
├── newproblem                  ← note generator script
└── Progress/                   ← SR state — GITIGNORED, never committed
    ├── Dashboard.md            ← derived aggregate: due-today + per-pattern proficiency + stats
    └── <Pattern>.md            ← per-problem SR rows for that pattern (source of truth)
```

- **Patterns** = the top-level folders (`Hashing`, `Heap`, `Backtracking`, `Bit Manipulation`,
  `Sliding Window`, `Two Pointers`, `1D Dynamic Programming`, …). Discover them by listing CWD
  dirs, excluding `Progress`, `.obsidian`, `.claude`, `.git`.
- `Progress/<Pattern>.md` is the **single source of truth** for each problem's SR state.
  `Dashboard.md` is **derived** — recompute it from the pattern files, never hand-maintain.

## Spaced-repetition schedule (Leitner, 5 boxes)

| Box | Interval | Meaning                    |
| --- | -------- | -------------------------- |
| 1   | 1 day    | just learned / just lapsed |
| 2   | 3 days   |                            |
| 3   | 7 days   |                            |
| 4   | 16 days  |                            |
| 5   | 35 days  | mastered (re-test monthly) |

- **Pass** (blind-solve correct, or MCQ correct): `box = min(box + 1, 5)`.
- **Lapse** (wrong / couldn't solve / major bug): `box = 1`.
- On every review: `last-reviewed = today`, `next-due = today + interval(box)`, `reps += 1`,
  `lapses += 1 if lapse else 0`.
- **Due** = `next-due <= today`. "About to be forgotten" = due, ranked by
  `overdue_days × difficultyWeight × (1 + lapses)` (difficultyWeight: easy 1, medium 1.3, hard 1.6).

Always compute dates from **today's actual date** (available in your context). Format `YYYY-MM-DD`.

## Workflow

### Phase 0 — Detect language

Detect the user's language from their message → `{LANG}`. All output and generated file content
in `{LANG}`. (Numbers, dates, badges, code stay universal.)

### Phase 1 — Discover & bootstrap

1. List pattern folders at CWD root (exclude `Progress`, `.obsidian`, `.claude`, `.git`).
2. Read `Progress/Dashboard.md` if it exists.
3. **First run (no `Progress/`)** → bootstrap:
   - Create `Progress/` and a `<Pattern>.md` per pattern folder (see Templates).
   - **Migrate** existing notes: for each problem note, read its frontmatter; if it still has
     `solved-1st/2nd/3rd`, convert to an SR row (see Migration), then **strip those three fields
     from the note's frontmatter** (leave `leetcode-id`, `difficulty`, `tags`, `memo`,
     `dg-publish`; optionally set `first-solved` = the old `solved-1st`).
   - Recompute `Dashboard.md`.
   - Confirm `Progress/` is gitignored; if not, tell the user to add `Progress/` to `.gitignore`.
4. **Subsequent runs** → also reconcile: any problem note not yet in a pattern file gets added as
   an untracked row (Box 0 / never solved).

### Phase 2 — Ask session type (MANDATORY AskUserQuestion)

Read the dashboard and build context-aware options. Always show counts. Include, as applicable:

1. **Due review** — `N` problems due/overdue today (most-overdue first). Include if N > 0.
2. **Drill weak pattern** — name the weakest pattern(s) (lowest avg box / most lapses).
3. **New from curriculum** — problems in `problemsets.toml` with no note yet, or never solved.
4. **Diagnostic** — sample across patterns, especially untested ones. Include if any ⬜ patterns.

User MUST select before proceeding. Never tag an option "(Recommended)".

### Phase 3 — Pick problems & mode

1. Pull candidate problems from the chosen pool (ranked per the SR weighting above).
2. Ask how to be tested (AskUserQuestion or infer from the session type), choosing among:
   - **Blind-solve** (default for review/drill) — 1–3 problems.
   - **MCQ quick-check** — 4 questions/round.
   - **Code-reading** — explain a stored snippet.
     Modes may be mixed (e.g. MCQ to triage, then blind-solve the weak ones).
3. For drill: also read the pattern file's `### Notes` to target the exact past confusion — re-test
   the same weakness from a **new angle**, don't repeat the identical question.

### Phase 4 — Run the test

Follow `references/quiz-rules.md`. Summary:

- **Blind-solve**: show the problem's title + difficulty + Description, **hide the Solution
  section**. Ask the user for approach and/or full code (their depth). This is conversational
  (NOT AskUserQuestion). Wait for their answer.
- **MCQ**: AskUserQuestion, 4 questions × 4 options, single-select, **zero hints**, randomized
  correct position. Types: pattern-recognition, complexity, key-insight, bug-spotting,
  output-prediction.
- **Code-reading**: show a snippet from the stored solution; ask the user to explain what it does /
  why it's correct / what an invariant is. Conversational.

### Phase 5 — Grade & explain

- **Blind-solve**: compare the user's logic & complexity against the stored Solution. Verdict
  **pass** (correct approach, right complexity, no fatal bug / handles edge cases) or **lapse**.
  Point out bugs, missed edge cases, and the canonical complexity concisely.
- **MCQ**: results table (Q / correct / user / ✅❌), one-line explanation for each miss.
- Map each item to its pattern.

### Phase 6 — Update state

For each problem reviewed:

1. Update its row in `Progress/<Pattern>.md`: new `box`, `last-reviewed=today`, `next-due`,
   `reps+1`, `lapses` (+1 on lapse). On a lapse, add/update a `### Notes` entry (the confusion +
   the key point).
2. Recompute `Progress/Dashboard.md` from all pattern files.
3. **Never touch the problem note's frontmatter** for review state.

## Templates

### `Progress/<Pattern>.md`

```markdown
# {Pattern} — SR Tracker

| Problem                           | ID  | Difficulty | Box | Last Reviewed | Next Due   | Reps | Lapses |
| --------------------------------- | --- | ---------- | --- | ------------- | ---------- | ---- | ------ |
| [[Hashing/0001-Two-Sum\|Two Sum]] | 1   | easy       | 3   | 2026-06-14    | 2026-06-21 | 4    | 1      |

### Notes

**Two Sum** — Confusion: 先存再查會用到自己。Key point: 查 `target-nums[i]` 要在存入 `nums[i]` 之前。
```

### `Progress/Dashboard.md`

```markdown
# LeetCode Review Dashboard

> Spaced-repetition tracker. Source of truth = `Progress/<Pattern>.md` (gitignored).
> Derived view — regenerated by the leetcode-tutor skill. Do not hand-edit.

## Due Today (N)

| Problem | Pattern | Difficulty | Overdue (d) | Box |
| ------- | ------- | ---------- | ----------- | --- |

## Proficiency by Pattern

| Pattern | Tracked | Avg Box | Lapses | Level | Detail |
| ------- | ------- | ------- | ------ | ----- | ------ |

> Level by avg box: 🟥 <1.5 · 🟨 1.5–2.5 · 🟩 2.5–3.5 · 🟦 >3.5 · ⬜ none

## Stats

- **Total tracked**: 0
- **Due today**: 0
- **Weakest pattern**: -
- **Strongest pattern**: -
```

## Migration (one-time, from old frontmatter)

For each note still carrying `solved-1st/2nd/3rd`:

- `reps` = number of filled `solved-*` slots; `lapses` = 0 (history unknown).
- `box` = `reps` capped at 3 (1 solve → box 1, 2 → box 2, 3 → box 3).
- `last-reviewed` = the latest filled `solved-*` date.
- `next-due` = `last-reviewed + interval(box)` (often already past → surfaces as due, which is
  correct — old problems should resurface).
- Then **remove `solved-1st/2nd/3rd` from the note frontmatter**; optionally keep
  `first-solved: <old solved-1st>` (written once, never updated).

## Important reminders

- ALWAYS read `references/quiz-rules.md` before crafting any question.
- **NEVER** write SR/review state into a problem note's frontmatter — it goes in `Progress/` only.
- `Progress/` is gitignored; `Dashboard.md` is derived — recompute, never hand-maintain.
- Blind-solve is the primary mode; MCQ is a fast supplement, never the whole story.
- Zero hints in MCQ options; randomize the correct position; no "(Recommended)".
- Always compute `next-due`/overdue from today's real date.
- Communicate in the user's language.
