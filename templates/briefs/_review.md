# <BATCH-ID> · <Title in plain words> — review brief

**Agent:** reviewer (<model>) · **Issued:** <date> · **Round:** <1 | re-review after fix-n>

## 1. What to review

| | |
|---|---|
| Repository and checkout | `<path>` — read here; run tests from your own fresh clone at `<scratch path>` |
| Branch under review | `batch/<BATCH-ID>` at `<sha>`, compared with `main` at `<sha>` (`git diff main...batch/<BATCH-ID>`) |
| Implementation brief(s) | `plan/briefs/<BATCH-ID>-implement.md` <and every sub-batch brief in this area> |
| Implementer report | `progress/<BATCH-ID>-implement.md` (final entry) |
| Progress log | `<workspace>/progress/<BATCH-ID>-review[-n].md` — create first, keep current; its final entry holds your full report |
| Fix brief (re-review only) | `plan/briefs/<BATCH-ID>-fix-<n>.md` — re-check only its findings |

## 2. Required checks

`plan/testing-discipline.md` is the current rule; where this brief is silent, follow it.

1. **Your runs; no gate.** From your fresh clone: build once, run the whole fast suites. **Repeat no run the
   implementer reported unless the repeat can catch something new.** Read each fail check in the implementer's log and
   judge whether the break hits the behavior the test claims; run one only where it is missing or misses its target, or
   a new break for a test you judge weak, and say which. Run tests the implementer did not that the change could break.
   For each run: command, summary, what it could catch.
2. **Scope.** Every scope and acceptance item present; nothing outside it changed.
3. **Design conformance.** Behavior against each cited section; cite section and file:line for every mismatch.
4. **Tests.** Would they fail if the code were wrong? Exact values, denied and negative cases, invariant violations,
   concurrency where relevant. Flag tautological or over-mocked tests.
5. **Rules.** <the project's non-functional rules relevant here>; patterns in `plan/performance-lessons.md` if it
   touches a list, job or query.
6. **Implementer assumptions.** Agree or dispute each, with reasons.
7. **Can it wait?** For each finding: fix now (gate or fresh clone broken, security or data integrity, user-visible,
   likely in normal use, a later batch builds on it, contradicts design others rely on, scaling cost with a known fix)
   or follow-up, and why.

## 3. Review focus for this batch

- **Standing:** local-only hardening beyond what the design requires; environment-supplied details hardcoded without
  being listed as integration points.
- <from the plan card and the orchestrator; state evidence as file:line at the reviewed SHA, never as an established
  defect>

## 4. Rules for the reviewer

- Edit, commit, stash or push nothing outside temporary probes in your own clone, which you restore and confirm clean.
- Evidence, not impressions: commands, output excerpts, file:line.
- Keep the tree still while anything runs. Batch your probes into one scripted pass. Fail-check any probe of your own
  that asserts an absence.
- Only one live stack (end-to-end environment) on the machine at a time; tear yours down; stop only what you started.

## 5. Report

The reviewer report (orchestrator instructions §10), with a verdict: `PASS`, `PASS WITH MINORS` or `CHANGES REQUIRED`.
