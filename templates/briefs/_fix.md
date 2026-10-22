# <BATCH-ID> · <Title in plain words> — fix brief <n>

**Agent:** implementer (<model>) · **Issued:** <date>

Checkout `<path>`, branch `batch/<BATCH-ID>` at `<sha>` (same as `plan/briefs/<BATCH-ID>-implement.md`, which gives
context). Review log: `progress/<BATCH-ID>-review.md`. Fix exactly the findings below; change nothing else.

**Progress log:** `<workspace>/progress/<BATCH-ID>-fix-<n>.md` — create first, keep current. Its final entry carries
your report, with any assumptions and questions verbatim.

| Finding | Severity | Location | What to change | Orchestrator's ruling (why now) |
|---|---|---|---|---|
| R-1 | | | | |

**Rejected findings (do not act on):** <id — reason>

**Answers to open questions:** <Q-n — answer>

<If a finding needs diagnosis: "Prove the cause before fixing, and say how you proved it." Or name the fix shape.>

Every test added or changed for a finding is shown to fail against the unfixed code, then pass; each check goes in the
log.

**Testing:** `plan/testing-discipline.md`. **No full gate**; the merge gate is the fix's full run. Build once, then the
tests for every finding, the touched module's tests and the whole fast suites, with each command and summary line in
the report. Keep the tree still while anything runs; batch fail-check probes into one pass.

Commit as `<BATCH-ID>: fix <n> — <what>` with the attribution lines from your dispatch, push the branch, and return the
implementer report covering only this fix.
