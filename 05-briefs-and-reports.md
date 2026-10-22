# 5. Briefs and reports

A brief is everything a cold agent needs to do one job. A report is everything the orchestrator needs to judge it.
Templates: [`templates/briefs/`](templates/briefs/).

## 5.1 Kinds of brief

| Brief | File | Agent | Purpose |
|---|---|---|---|
| Implement | `plan/briefs/<id>-implement.md` | Implementer | Build one batch or sub-batch |
| Review | `plan/briefs/<id>-review[-n].md` | Reviewer | Independent validation, findings, verdict |
| Fix | `plan/briefs/<id>-fix-<n>.md` | Implementer (or the reviewer, sent back) | Fix exactly the accepted findings |
| Triage | `plan/briefs/<id>-triage.md` | Reviewer | Reproduce a gate failure, root-cause it, report; no edits |
| Gate | `plan/briefs/<id>-gate[-machine].md` | Runner | Run the merge gate on one exact commit and report observations |

## 5.2 What makes a good brief

- **Where to work, exactly.** Repository, absolute checkout path, branch and the SHA it was cut from, progress log path,
  ports and database name for a parallel worktree. "If the branch or tree is not as stated, stop and report."
- **Goal in one paragraph**: what exists afterwards and why later batches need it.
- **Read first, in order**: the repository's agent rules, the design sections, the files from earlier batches to build
  on or copy (contracts, fixtures, patterns). **Point to files; do not paste them.**
- **Scope in, and scope out.** "Out" lists what a reader might think is included. An implementer builds all of the
  scope and nothing outside it.
- **Decisions already made**: resolved ambiguities and decision ids, so the agent neither re-derives nor re-asks them.
- **Acceptance**: the definition of done plus the card's items, made concrete (exact demonstrations to paste).
- **Testing**: what to run, what not to run (no full gate for implementers), fail checks required, and a pointer to the
  testing-discipline file for anything the brief does not say.
- **Known pitfalls** from the ledger notes, follow-ups and flake register in this area, plus the standing ones:
  humble stand-ins, laziest thing that works (shortcuts reported, not commented), a test must be able to fail.
- **Report shape** and commit rules: message format, push after each commit (an unpushed commit exists on one
  machine), never merge.
- **Language rules** for any user-facing text (e.g. the owner's preferred spelling, stated in every content brief).
- For a fix that turns on **diagnosis**, say so and ask for the cause to be proved before the fix, or name the fix
  shape; stating only the symptom invites a fix that clears the error rather than the cause.

What a brief must not do: assert a defect as established without having seen it in the tree; invite a runner to
diagnose; decide open behavior on the agent's behalf without a recorded decision.

### Who writes briefs

A design-brief writer agent drafts implement briefs, so the reading stays out of the orchestrator's context. It reads
the card, backlog rows, decisions, design, testing rules, performance lessons and the current code, writes the brief and
a final section **"Questions for the orchestrator"**, each with options and a recommendation. It never decides open
behavior. The orchestrator answers, edits and issues. Related briefs go to one writer, which reads the shared material
once.

Review, fix and gate briefs are short and the orchestrator writes them from the templates. Write each brief whole with
a file-writing tool; do not generate briefs by scripted string replacement (on the reference project, Windows paths turned into control
characters that way).

## 5.3 The implementer report

Nine items, in this order, also written as the final entry of the progress log:

1. **Summary.**
2. **Changes by area**, with paths.
3. **Tests added, by layer** (unit, integration, architecture, component, end-to-end), with the **fail check** for each
   key test: the line broken, the failing test and its assertion output, the restore.
4. **What was run**: exact commands, results, counts.
5. **Assumptions `A-1..A-n`**: what, why, design reference. The full list, verbatim.
6. **Questions and blockers `Q-n`.**
7. **Design text found wrong or contradictory**, with suggested wording.
8. **Follow-ups outside scope**, including deliberate shortcuts with their ceiling, and integration points assumed.
9. **Commits** (SHAs) and the pushed branch.

## 5.4 The reviewer report

Six items:

1. **Verdict**: `PASS`, `PASS WITH MINORS` or `CHANGES REQUIRED` (any blocker or major means changes required).
2. **What was run**: commands, results, counts, and for each run what it could catch that the implementer's could not.
3. **Findings**, each with:

| Field | Content |
|---|---|
| Id | `R-n` |
| Severity | `blocker`, `major`, `minor`, `nit` |
| Location | `file:line` read in the checkout |
| Finding | One sentence |
| Evidence | Command and output excerpt, or the code |
| Design reference | Section or requirement id |
| Suggested fix | Smallest change |
| Can it wait? | Fix now (which reason) or follow-up, and why |

4. **Scope check**: missing, extra.
5. **Implementer assumptions disputed**, with reasons (agree with the rest explicitly).
6. **Each review focus item answered**, including each implementer fail check judged (and any the reviewer ran).

Evidence over impressions: a finding cites a line the reviewer read or output it produced, never a search-index answer
or the implementer's say-so.

## 5.5 Id conventions

| Prefix | Meaning | Scope |
|---|---|---|
| `A-n` | Implementer assumption | Per report |
| `Q-n` | Question or blocker | Per report or brief |
| `R-n` | Review finding | Per review |
| `D-n` | Design text found wrong (optional) | Per report |
| `IMP-n` | Decision log entry | Project-wide, never reused |

Cite them with the batch: "B12 R-3", "B7a A-1".

## 5.6 How the orchestrator reads a report

1. **Read the log's final entry**, not a chat summary.
2. **Check the claims that matter against the commit**: the pushed SHA exists on the branch; any value or literal you
   will rely on (`git grep` at the SHA); the fail checks are present and each break plausibly hits the behavior the
   test claims.
3. **Check scope**: every acceptance item addressed; nothing unexpected in the file list.
4. **Rule on assumptions**: accept, or turn into a review focus item; any that changes behavior is a question.
5. **Answer questions** from the design and decisions log if possible; otherwise ask the owner and mark the batch
   `blocked`, while other lanes continue.
6. **Collect design corrections** for the design edit at merge.
7. **Gate claims**: only the gates log line counts. A runner's "exit 0" without its line means the gate has not
   finished.

For a reviewer report, adjudicate every finding (accept, reject with reason, design question) and every disputed
assumption; then write the fix brief or proceed to the merge gate ([03 §11](03-batch-lifecycle.md)).

Treat output from an agent outside your session (another model, another tool) with extra care: on the reference project one
external model's pasted fail checks were partly not literal, so its reviews were told to check each pasted check against the
code and re-run any that did not match.
