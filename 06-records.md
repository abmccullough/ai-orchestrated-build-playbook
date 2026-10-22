# 6. Records

What goes where. The rule behind all of it: **state lives in files, in git, not in anyone's memory or chat.** A cold
session, another workstation or the owner the next morning must be able to pick up from the files alone.

## 6.1 The files

| File | Holds | Who writes | When |
|---|---|---|---|
| `plan/orchestrator-session.md` | Session log: the latest HANDOFF at the top (running agents, owed items, state checked against git, standing rules), dated entries below | Orchestrator | Every step; a handoff before a fresh session ([11](11-context-management.md)) |
| `plan/ledger.md` | One row per batch; §Follow-ups; §Flake register; accepted ceilings | Orchestrator only | Every step |
| `plan/decisions.md` | Decisions `IMP-n`: what was chosen, why, what it rules out | Orchestrator only | Whenever a gap is resolved |
| `plan/testing-discipline.md` | **The current testing rules** | Orchestrator only | With every testing decision or lesson, same commit |
| `plan/performance-lessons.md` | Performance patterns found and fixed, with instances | Orchestrator only | At every merge that fixes a performance defect and every measurement |
| `plan/integration-points.md` | Enterprise-supplied details stood in, local-only choices, rules that differ outside dev | Orchestrator, from reports | At merge |
| `plan/run-plan-<name>.md` | An active run's batches and its "Status — resume here" table | Orchestrator | Every step, with the ledger |
| `plan/briefs/*.md` | Templates and every issued brief | Orchestrator, brief writer | Before dispatch |
| `progress/<brief-name>.md` | One log per dispatch: status block, steps, final report | The dispatched agent | Every step |
| `progress/gates.log` | One line per gate, appended by the gate wrapper script only | The script, never an agent or person | Each gate end |
| `design/**` | The authoritative design | Orchestrator only | Same batch as the behavior change |

Commit workspace changes **by path** (`git add <paths>; git commit`), never `commit -a`: running agents keep
uncommitted work in the workspace (progress logs, scripts), and a blanket commit claims it. (On the reference project, an
orchestrator commit swept an agent's half-finished script edit.)

## 6.2 The ledger

Header: who edits it, the status values, verdict values, the current phase and `main` SHAs.

Batch rows:

| Batch | Title | Repo | Status | Branch | Verdict | Loops | Merge SHA | Notes |
|---|---|---|---|---|---|---|---|---|
| B12 | Order export | app | merged | batch/B12 (deleted) | CHANGES REQUIRED (review 1); fix 1 accepted without re-review | 1 | 1a2b3c4 | Gate line; findings accepted/rejected with reasons; follow-ups created; design edits |

Notes are terse but complete: the gate line (or its key fields), each finding's ruling, decisions taken, what the next
gate or sweep must cover if this one was narrowed. Rows get long; that is fine, because the ledger is read by grep, not
end to end.

**§Follow-ups**: one row per deferred item: source (batch and finding id), what, why it can wait, ceiling and upgrade
path if a shortcut, the batch that should pick it up, status. Close rows when done, with the batch that did it.

**§Flake register**: every test that failed and then passed alone, outside the batch under test: test, date, gate,
error text, status. It has teeth: every entry is scheduled for a fix, and one that fires twice is not tolerated.

**Accepted ceilings** (optional appendix): limits deliberately accepted, with the measurement and the trigger to
revisit.

## 6.3 The decisions log

Each entry: `## IMP-n — title`, date and who decided (the owner, or "orchestrator; for review"), the decision, *Why*,
and *Revisit* (what would make you change it). Rules for the file:

- It records the decision and why, not the specification; once shipped, the code and tests are the spec.
- **Numbers and sub-clause numbers are never reused or renumbered.** Code, briefs and frozen migrations cite them. A
  superseded entry or clause stays and points at what replaced it.
- Keep a line at the top: "Still open for the owner's review: …", and clear it with the owner.
- Testing decisions also edit the testing-discipline file in the same commit; that file wins if they disagree.

*Why separate the testing file:* on the reference project the decisions log grew to hundreds of kilobytes, and the owner said decisions "get lost"
there. Within an hour of writing the testing file, it caught a retired rule still being applied from an old entry.

## 6.4 Testing-discipline and performance-lessons files

Both are **living current-rule files**, consulted before every brief, gate, review or test ruling (testing) and every
brief or review touching a list, a job or a query (performance). Nothing about testing lives only in a decision, a
ledger note or a brief: a ruling, a lesson from a gate or a triage, a new mechanic lands in the file in the same commit,
or it is lost. Skeletons in [`templates/`](templates/).

## 6.5 Progress logs

One per dispatch, named after its brief (`<id>-implement.md`, `<id>-review.md`, `<id>-fix-1.md`, `<id>-gate.md`).
Status block on top, timestamped entries, final report in full. To know where any agent is, read its log. Shape:
[02 §2.4](02-roles-and-agents.md).

## 6.6 The gates log

One line per gate, written only by the gate wrapper: timestamp, exit code, duration, `clean`/`DIRTY`, full commit SHA,
checkout, output file path, lanes run (with markers for narrowed runs), and any machine-sleep time. It is the only
evidence a gate ran. No agent writes, edits, "corrects" or redirects output into it. (On the reference project, a runner once appended a
hand-written line; another redirected gate output into the file and destroyed its history.)

## 6.7 Batch ids

- **Check the id is unused** before naming a batch: grep the ledger and `plan/briefs/`. On the reference project a new brief overwrote a
  merged batch's brief because the id was reused; if a file write reports "updated" for a file meant to be new, stop.
- **A letter suffix for sub-batches** (`B10a`, `B10b`), a `p` suffix for the pipeline half, an `m` suffix for a
  measurement: whatever you choose, write it in the plan's "reading a batch card".
- **Always name a batch in plain words beside its id** to the owner and in run-plan tables: "B14 (data-growth
  fixes)". Nobody memorizes ids.

## 6.8 Integration-points register

Every enterprise-supplied detail filled with a stand-in, every local-only choice, every shipped rule that behaves
differently outside development. Implementer reports list the points their batch assumed; the orchestrator adds them
at merge. It is the handoff document for whoever wires up the real environment.
