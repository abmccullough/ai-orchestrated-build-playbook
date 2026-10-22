# <BATCH-ID> · <Title in plain words> — implementation brief

**Agent:** implementer (<model>) · **Issued:** <date> · **Plan card:** `plan/execution-plan.md` <BATCH-ID> (or run plan)

## 1. Where to work

| | |
|---|---|
| Repository | `<repo>` |
| Checkout (run every command here) | `<absolute path: repos/<repo> or .worktrees/<repo>--<BATCH-ID>>` |
| Branch | `batch/<BATCH-ID>` (already created from `main` at `<sha>`) |
| Ports / database name | `<parallel batches only>` |
| Design root | `<workspace>/design/` — read, never edit |
| Progress log | `<workspace>/progress/<BATCH-ID>-implement.md` — create first, keep current. **Its final entry carries your full report, every assumption and question verbatim** |

Never touch another checkout. If the branch or tree is not as stated, stop and report.

## 2. Goal

<One paragraph: what exists after this batch and why later batches need it.>

## 3. Read first

1. `<repo>/<agent rules file>`
2. <design sections, in reading order>
3. <files from earlier batches to build on: contracts, fixtures, patterns to copy>

## 4. Scope

**In:**
- <deliverable>

**Out (do not build):**
- <things a reader might think are included>

## 5. Decisions already made

- <resolved ambiguities, decision ids, orchestrator answers — so you neither re-derive nor re-ask them>

## 6. Acceptance

- The definition of done in `plan/execution-plan.md`.
- <the card's acceptance items, made concrete; demonstrations whose literal output you paste>

## 7. Testing

`plan/testing-discipline.md` is the current rule; where this brief is silent, follow it.

- **Targeted, not the full gate.** Build with warnings as errors; the whole fast suites; integration tests filtered to
  the modules you touched and the modules consuming their contracts: `<filters>`; front-end checks if you change the
  front end; your fail checks. Put every command and its summary line in the report.
- **Full gate or end-to-end only if you judge it necessary, and say why** — the full gate runs once at the merge.
  Command, for that case: `<gate wrapper command>`.
- **Fail checks in the progress log in full:** the line you broke, the failing test and its assertion output, the
  restore. The reviewer judges them from the log alone.
- Keep the tree still while anything runs. Commit first, then test.

## 8. Known pitfalls

- **Humble stand-ins.** Do not invent production designs for environment-supplied details; use the simplest stand-in
  that exercises the shipped code and list it as an integration point.
- **Laziest thing that works.** No code, then what the repository has, then the standard library, then an installed
  dependency, then the shortest thing that works. Never for trust-boundary validation, data-loss handling, security or
  accessibility. A deliberate shortcut with a ceiling: name the ceiling and upgrade trigger **in the report, not a code
  comment**.
- **A test must be able to fail.** A test that passes against broken code is not done.
- <from the ledger notes, §Follow-ups and §Flake register for this area; performance patterns if it touches a list,
  job or query>
- <language rules for user-facing text, e.g. US or UK spelling>

## 9. Report

Return the implementer report (orchestrator instructions §10), also as the final log entry. Paste the literal output of
every demonstration the acceptance section asks for. Commit on `batch/<BATCH-ID>` with messages `<BATCH-ID>: <what>`
and the attribution lines from your dispatch. **Push after each commit.** Do not merge.

## Questions for the orchestrator

<Brief writer only; removed or answered before issue. Each: the question, options, recommendation.>
