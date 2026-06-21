# Quiz Design Rules (LeetCode)

The vault tests a **procedural** skill (solve / recall / implement), so blind-solve is primary.
MCQ and code-reading are supplements. Each format has its own rules.

## Mode A — Blind-solve (primary, free response)

1. Show: problem **title + difficulty + Problem Description**. **Hide the Solution section.**
2. Ask the user for either (a) the approach in words, or (b) full code — let them choose depth, or
   ask "思路還是直接寫 code？".
3. This is **conversational** — never use AskUserQuestion for blind-solve.
4. **Grading rubric** (compare against the stored Solution):
   - **Approach correct?** Does the algorithm actually work for all inputs the problem allows?
   - **Complexity correct?** Did they state the right time/space, and does their code match it?
   - **Bugs / edge cases?** Off-by-one, empty input, duplicates, overflow, the specific trick the
     canonical solution relies on.
   - **Verdict**: `pass` = correct approach + right complexity + no fatal bug. Otherwise `lapse`.
5. After grading, show the canonical solution and name the one key insight in a sentence.

## Mode B — MCQ quick-check (AskUserQuestion)

### Zero-Hint Policy (CRITICAL)

Answerable ONLY by someone who actually knows the material.

1. **Option descriptions NEVER reveal correctness.**
   - BAD: label `O(n)`, desc "optimal because the deque is monotonic"
   - GOOD: label `O(n)`, desc "linear in the array length"
2. **No "(Recommended)"** on any option.
3. **Randomize** the correct position — never always first/last.
4. **Question phrasing** asks about behavior/output/choice, never hints the answer.
5. **Plausible distractors** = real concepts that represent common misconceptions
   (e.g. `O(n log n)` vs `O(n)` for a monotonic-deque sliding window).

### MCQ question types

1. **Pattern recognition**: given a problem description, "which technique applies?"
   (sliding window / two pointers / monotonic stack / hashing / heap / binary search / DP / …)
2. **Complexity**: "optimal time/space of this problem?"
3. **Key insight**: "what invariant / trick makes the optimal solution work?"
4. **Bug spotting**: show a near-correct snippet, "where is the bug?"
5. **Output prediction**: "what does this code return for input X?"

### Format

- 4 questions/round, 4 options each, single-select.
- Header ≤ 12 chars, e.g. `Q1. Pattern`, `Q2. Big-O`.

## Mode C — Code-reading (free response)

Show a snippet from the stored solution. Ask the user to explain what it does, why it's correct,
or to name an invariant. Conversational. Grade on whether they show real understanding vs surface
memorization.

## Difficulty balancing

- **Diagnostic**: easy 40% · medium 40% · hard 20%.
- **Weak-pattern drill**: medium 30% · hard 70%; bias toward problems with `box = 1` or `lapses ≥ 1`.
- **Due review**: whatever is due — span difficulties as they fall.

## Drilling lapsed problems

When re-testing a problem that previously lapsed (read its `### Notes`):

- Do NOT repeat the identical question — re-test the **same weakness from a new angle**.
- E.g. if they forgot to store index (not value) in a monotonic deque, give a variant where the
  same indexing mistake would surface, or ask them to predict the output of the buggy version.

## SR update protocol (after grading)

For every problem reviewed:

1. Update its row in `Progress/<Pattern>.md`: `box` (pass → `min(box+1,5)`, lapse → `1`),
   `last-reviewed = today`, `next-due = today + interval(box)`, `reps += 1`, `lapses += lapse`.
2. On a lapse, add/update a `### Notes` entry: `Confusion:` … `Key point:` …
3. Recompute `Progress/Dashboard.md` (due-today list + per-pattern avg box + stats).

## Language

All output and file content in the user's detected language. Code, Big-O, dates, badges universal.
