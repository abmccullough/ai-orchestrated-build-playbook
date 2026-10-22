# 3. The batch lifecycle

From an idea to a merged, recorded batch. Steps 1–4 happen once per feature or phase; steps 5–15 once per batch.

```
design → refine → plan → design edits → brief → implement → read report → review → adjudicate
      → fix loop (as needed) → merge gate → merge → record → (phase end) checkpoint sweep
```

## Ledger statuses

| Status | Meaning |
|---|---|
| `pending` | In the plan, not started |
| `implementing` | Implementer (or sub-batch implementers) dispatched |
| `reviewing` | Reviewer dispatched |
| `fixing` | A fix brief is out |
| `blocked` | Waiting on the owner (a behavior question, an approval, a fix with no clear solution) or an external event |
| `merged` | On `main`, with merge SHA, verdict, loops and notes |

Review verdicts: `PASS`, `PASS WITH MINORS`, `CHANGES REQUIRED` (any blocker or major finding).

Update the ledger, and the active run plan's status table and the session log, **after every step**, in the same
commit.

---

## Before batches: design and plan

1. **Design.** A feature starts as a design draft or a section of the design ([04](04-design-and-planning.md)).
2. **Refine.** The owner answers the draft's decision table; the orchestrator resolves what the design already answers.
3. **Plan.** Cut it into batches in the execution plan or a run plan: dependencies, parallel eligibility, acceptance,
   review focus.
4. **Design edits.** The orchestrator moves the accepted draft into `design/` (the authoritative copy) before the first
   batch is briefed. Later behavior changes are edited in the batch that makes them.

## Per batch

### 5. Pick the batch

The first in plan order (or priority order, see [10](10-working-with-the-owner.md)) whose dependencies are all
`merged` and that is not `blocked`. Check its id is unused before naming a new one.

### 6. Prepare the brief

A design-brief writer drafts `plan/briefs/<id>-implement.md` from the template, the card, the backlog rows, the design
and the current code. The orchestrator answers the "Questions for the orchestrator" (from the design and decisions if
it can; otherwise the owner), edits what needs it, and finishes it. It skims the cited design only enough to judge the
questions. Several related briefs go to **one** writer in sequence. Details: [05](05-briefs-and-reports.md).

### 7. Branch

Serial on the primary checkout (`batch/<id>` from fresh `main`), or a worktree for a parallel batch. One branch per
repository the batch touches. Details: [07](07-branches-worktrees-merging.md).

### 8. Implement

Dispatch the implementer with the brief path and progress log path, plus a two-line summary. Ledger `implementing`.
Wait for the notification; never predict or summarize a report that has not arrived. Large or multi-area batches go
as sub-batches, each its own fresh dispatch on the same branch.

### 9. Read the report against the brief

- Every scope and acceptance item claimed? Every command and count present? Fail checks shown in the log?
- Assumptions `A-n`: accept, or send to the reviewer as a focus item. Any that changes behavior is a question.
- Questions `Q-n`: answer from the design and re-dispatch, or ask the owner and mark `blocked`.
- Design text the implementer found wrong: note for the design edit at merge.
- Any value or literal you will rely on: read it from the commit, not the report.

### 10. Review

Reviewer brief from the template: the implement brief, the report (or its log), branch and checkout, the card's review
focus plus your own. Ledger `reviewing`. The reviewer runs **no full gate**: it builds a fresh clone, runs the fast
whole-project suites, reads the implementer's fail checks rather than repeating them, runs tests the implementer did
not, and repeats nothing unless the repeat can catch something new ([08](08-testing-discipline.md)).

Do not assert a defect as established in a review brief unless you have seen it in the tree; it steers the reviewer and
wastes the independent look.

### 11. Adjudicate

For each finding: **accept**, **reject** (reason in the ledger notes), or **design question** (to the owner). Rule on
every implementer assumption the reviewer disputes. Resolve discrepancies; do not re-review the whole diff yourself.

**Fix now only with a reason.** A finding goes into a fix brief only when at least one holds:

| Fix-now reason | Example |
|---|---|
| Breaks the gate or a fresh clone | A file missing from the push |
| Security or data-integrity defect | A write path without an authorization check |
| Users will see it | Wrong label, broken layout |
| Likely in normal use | Fails on the common path, not a rare interleaving |
| A later batch builds on it, or it gets harder to fix later | A contract shape |
| Contradicts design text others rely on | An event name |
| A scaling cost with a known fix | A per-row query on a list |

Otherwise it becomes a **§Follow-ups row** in the ledger (or a design backlog row for deferred scope), whatever its
severity, naming it and why it can wait. The fix brief's ruling column states the reason for each fix taken.

### 12. Fix loop

Accepted findings go into `plan/briefs/<id>-fix-<n>.md` for an implementer (fresh, or warm if it qualifies) on the same
branch. Ledger `fixing`. Every test added or changed for a finding is shown failing on the unfixed code, then passing.

**Re-review in proportion:** a narrow fix (a few files, each finding's fail check shown) gets **no re-review**, runtime
change or not. The orchestrator reads its diff, confirms the pushed commit and the fail checks, greps for readers of
anything removed, and the merge gate is its full run. **Still re-reviewed:** anything touching security,
authorization, data integrity, schema migrations or seeding, or a broad fix. A re-review checks only the fixed
findings.

**When to stop:** never on a count. Make every clear, material fix. Bring the batch to the owner only when there is no
clear solution (cause not understood, a fix failed with no new direction, or a behavior decision). A fix that turns on
diagnosis (a flaky or order-dependent test, several plausible causes) names the fix shape in the brief or asks the
agent to prove the cause before fixing.

### 13. Merge gate

1. Confirm no agent is active in the batch's checkout.
2. `git fetch origin` **first**.
3. Merge `origin/main` into the batch branch (merge commit, never rebase); regenerate generated files rather than
   hand-editing conflicts; push.
4. A runner gates **that commit** from a fresh clone, with the end-to-end lane when the batch touched something only a
   live stack exercises. One full gate per machine at a time.
5. Read the result from the gates log line and the output file, never from the runner's prose.

Combine ready batches into one gate where it makes sense. A failing gate is **triaged, never re-run to get past it**.
Details: [07](07-branches-worktrees-merging.md), [08](08-testing-discipline.md).

### 14. Merge

On green: on `main`, `git merge --no-ff origin/batch/<id>`; `git diff <gated SHA> main` must be empty; push `main`;
remove the worktree; delete the branch.

### 15. Record

- Ledger row: `merged`, merge SHA, verdict, number of fix loops, notes (gate line, accepted and rejected findings,
  follow-ups created).
- Decisions taken → decisions log. Testing rulings → testing-discipline file. Performance fixes → performance-lessons
  file.
- Design edits for any behavior the batch changed or found wrong (same batch, cite its id).
- Cross-repository pins bumped if the batch changed something another repository consumes.
- Run plan status table and session log updated. Workspace committed **by path** and pushed.

## Phase end: checkpoint

At the end of each phase:

1. **Regression sweep** by a runner: every repository's full gate from fresh clones of `main`, with end-to-end; any
   determinism checks (e.g. seed twice and compare); slow sign-off tests. Reuse a green merge gate when `main` is
   exactly that commit.
2. **Visual pass** in a real browser where the phase shipped screens; it finds things no test sees.
3. **Report to the owner**: batches merged (in plain words), decisions taken, decisions flagged for review, open
   questions, known risks, next phase.
4. Push the workspace.

The reference project's owner later dropped the wait at phase ends ("it is arbitrary anyway"): sweep, report, carry on. Agree with your
owner which they want; hard stops still hold either way.
