# <BATCH-ID> · <Title in plain words> — triage brief

**Agent:** reviewer (<model>), triage only: **no edits, no fix** · **Issued:** <date>

## What failed

- Gate line (verbatim from `progress/gates.log`): `<line>`
- Output file: `<path>`; failing test(s): `<names>`; failure text: see the output file at `<line range>`.
- The failing tests re-run alone on the same tree: `<reproduced | passed>` (`<command>`, `<result>`).
- What the batch changed: `git diff <main sha>...<gated sha> --stat` (read it yourself).

## Do

1. Create `<workspace>/progress/<BATCH-ID>-triage.md` first; keep it current.
2. Make a fresh clone of `<gated sha>` at `<scratch path>`.
3. Reproduce with targeted runs (if it passes alone, try the test's whole folder or a second build path; order and
   shared data are the usual causes).
4. Find the root cause. Read the full error and its inner chain before theorizing. Say how you proved it: a minimal
   change that makes it pass or fail on demand, restored afterwards.
5. Say whether the cause is in the batch's own change, pre-existing on `main`, or the test itself.

## Report

Cause in one paragraph; evidence (commands, output, file:line); whether it is the batch's, `main`'s or the test's; the
smallest fix and where; what else could be affected (callers, sibling tests with the same pattern). No edits.
