# Ledger

The build's state. Only the orchestrator edits this file. Read it by search (grep the batch id), not end to end.

Statuses: `pending` · `implementing` · `reviewing` · `fixing` · `blocked` · `merged`.
Verdicts: `PASS` · `PASS WITH MINORS` · `CHANGES REQUIRED`.

**Current phase:** <n> · **`main`:** <repo> `<sha>`, <repo> `<sha>` · **Cross-repository pins:** <pin> = `<sha>`

## Batches

| Batch | Title (plain words) | Repo | Status | Branch | Verdict | Loops | Merge SHA | Notes |
|---|---|---|---|---|---|---|---|---|
| 0.1 | Repositories created and cloned | all | pending | — | — | — | — | |
| <F01> | <title> | <repo> | pending | | | | | <brief path; dispatch time (clock read); implementer report summary; review verdict; each finding's ruling (accepted / rejected: reason / follow-up FU-n); fix loops; gate line; skips and their reasons; design edits; decisions> |

## Follow-ups

Deferred work found during batches. Close a row with the batch that did it.

| Id | Source (batch, finding) | What | Why it can wait / ceiling and upgrade trigger | Pick up in | Status |
|---|---|---|---|---|---|
| FU-1 | <A01 R-4> | | | | open |

## Flake register

Every test that failed and then passed alone, outside the batch under test. Every entry is scheduled for a fix; one
that fires twice is not tolerated.

| Test | Date | Gate (SHA, machine) | Error text (or output file and line) | Status / fix batch |
|---|---|---|---|---|

## Accepted ceilings

Limits deliberately accepted, with the measurement and the trigger to revisit.

| Ceiling | Measured | Revisit when | Decision |
|---|---|---|---|
