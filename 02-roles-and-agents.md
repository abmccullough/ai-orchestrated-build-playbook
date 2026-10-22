# 2. Roles and agents

One orchestrator session, four agent types, one human owner. Separation of duties is the point: the agent that writes
code never judges it, the agent that judges it never fixes it in the same pass, and the agent that runs the gate draws
no conclusions.

## 2.1 Who does what

| Role | Model (reference implementation) | Does | Never |
|---|---|---|---|
| **Owner** (human) | — | Decides behavior, priorities, autonomy grants; approves irreversible actions; reviews at checkpoints | Needs to read transcripts; the files carry the state |
| **Orchestrator** | Strongest available (Opus) | Plans, writes and finishes briefs, dispatches, reads reports, adjudicates findings, merges, records, edits `plan/` and `design/`, talks to the owner | Writes product code or tests; fixes findings itself; merges without a review pass; lets an agent decide open behavior |
| **Implementer** | Opus | One brief: code, tests, migrations, fixtures; fail checks; commits and pushes the batch branch; reports | Pushes `main`; edits `plan/` or `design/`; guesses at a behavior gap; weakens a test to pass |
| **Reviewer** | Opus | Independent validation against brief and design; findings with evidence; verdict. Also takes **triage** briefs (reproduce, root-cause, report) | Edits anything on a review or triage brief; fixes a finding in the dispatch that found it |
| **Design-brief writer** | Opus | Drafts implement briefs from the plan card, backlog rows, design and current code; lists questions with recommendations | Decides open behavior; edits ledger, plans, decisions or design; runs tests, builds or containers |
| **Runner** | Small, cheap model (Haiku) | Mechanical, fully specified work: gates, version-pin bumps, regeneration, a specified rename | Design or scope judgments; stating a cause; writing to the gates log |

Template definitions: [`templates/agents/`](templates/agents/).

### Why the orchestrator writes no code

Its context is the scarcest resource in the system: it holds the plan, the open decisions and every batch in flight.
Code-level work fills it with detail it does not need to keep. It also preserves independence: an orchestrator that
fixed a finding would be judging its own work at merge.

### Why the reviewer never fixes what it found

The orchestrator rules on every finding first (some are rejected, some deferred), and an independent gate checks the
result. A reviewer that fixes in the same pass collapses review, adjudication and fix into one unchecked step. A
reviewer may be sent back later with a separate **fix brief**, and then follows the implementer's rules.

### Why the runner states no cause

A runner sees one command's output and none of the context: which files a batch touched, when code last changed, what
already ran green. On the reference project two checkpoint runners did their mechanical work correctly and then
reported dramatic wrong conclusions (non-determinism that was expected bookkeeping; a regression blamed on three batches
that had touched no relevant file, when the real defect was much older). A wrong cause stated confidently sends the fix to the wrong place.
Runners report "test X failed with this message", nothing more.

## 2.2 Model choice

- **Implement and review on the strongest model.** The reference project ran a recorded experiment (tokens, tool calls, wall time,
  verdicts, findings accepted, later misses) comparing Opus at medium effort with Sonnet. The owner's cost analysis
  found Opus more efficient overall: fewer fix loops and shorter runs outweighed the per-token price. From then on every
  implementer, fix and reviewer dispatch used the Opus definitions.
- **Mechanical work on a cheap model.** Gates, pin bumps and regeneration need no judgment; a small model is fine as
  long as its brief is fully specified and its report is read against the gates log, never on its word.
- **Run your own comparison** if prices or models change: a model-experiment table per dispatch, kept for a week, is
  enough to decide.
- Agent harnesses may not take a per-dispatch effort setting; the reference project used separate definitions per model and effort.

## 2.3 Agents start cold

Every agent starts with no conversation context. Therefore:

- **Briefs are self-contained.** They name the checkout, branch, progress log, design sections, decisions already made,
  scope in and out, acceptance, testing rules and report shape.
- **Briefs point to files; they do not paste them.** The agent reads the design section itself; pasting duplicates
  context and goes stale.
- **No agent for a single read or command.** Spawning costs a cold start; do one-off lookups directly.
- **Read the progress log, not the transcript**, to know where an agent is.

## 2.4 Progress logs are the resume point

Every dispatch names a progress log (`progress/<brief-name>.md`). The agent:

1. creates it first: batch, brief, checkout, branch, starting SHA, start time (read from the clock);
2. keeps a **Status** block at the top current: last completed step, next step, anything in flight (uncommitted
   changes, a temporary edit not yet reverted, a scratch clone, a running server or container);
3. appends a timestamped entry per completed step (not per command);
4. logs a temporary edit before making it and again when reverted;
5. ends with the full report, including the **complete** assumption list and questions verbatim.

*Why the full report goes in the log:* on the reference project a reviewer could rule on only 8 of 12 assumptions because the report
arrived from another machine as a chat message and the log had only the first four. The log is in git; it crosses
machines and sessions.

## 2.5 Resuming: fresh agent or the old one

- **Default: resume the work, not the agent.** After any pause, usage stop or new session, dispatch a fresh agent of
  the same type with the same brief and "resume from the progress log at `<path>`". It compares the checkout with the
  log, notes differences and continues from the next step. Waking an old agent reloads and re-caches its whole context,
  which is expensive.
- **Exception: a warm fix.** Resume the batch's earlier implementer for a fix only if it finished within the prompt
  cache lifetime (reference implementation: one hour) and its run was ordinary. A bloated context is worse than a cold start. When unsure,
  go fresh. Write fix briefs self-contained so either choice works.
- **A message queued to a running agent is owned by that agent**, even if it reports "done" before reading it: it will
  resume and act on it. Before dispatching anyone else into its checkout, check `git status` and the tail of its log.
- **A dead agent sends no notification.** On a long unattended run, check heartbeats (log file growth, gate output
  growth, containers alive) and treat two consecutive checks with no change as halted.

## 2.6 Concurrency

- **Serial by default.** Two agents never share a checkout.
- **A hard agent limit**, set with the owner from RAM and usage. Reference implementation: three concurrent agents on 16 GB (each worktree
  builds and runs its own test database). Batches already running finish; nothing new starts at the limit.
- **Throttle near usage limits.** Reference implementation rule: above about 80 % of the short-window usage, drop to one agent so nothing
  stalls mid-batch for hours.
- **Lane priority when slots are contended**, agreed with the owner (reference implementation: the pipeline lane first because it was the
  longer path).
- **One full gate per machine at a time.** A running gate does not hold an implementer or reviewer; only a second gate
  waits.

## 2.7 Parallel only when context does not overlap

Parallel agents that must read the same design sections and code each pay to load them and cannot share what they
learn. Give overlapping work to **one agent in sequence** instead.

- Three briefs for batches over the same screens: one design-brief-writer dispatch covering all three, not three.
- Sub-batches of one area: sequential implementers on one branch, one reviewer.
- Real parallelism: batches the plan marks parallel, in different areas, each in its own worktree.

## 2.8 When the orchestrator is idle

If every in-flight batch is waiting on an agent and the orchestrator has nothing of its own to do, it stops with a
one-line status of what is awaited. It does not fill the wait with speculative reading for a batch whose inputs depend
on a pending report. *Why:* idle groundwork bloats the orchestrator's context and forced a mid-build compaction on
the reference project. A short wait can be a compaction point; a large session is better handed off to a fresh one
([11-context-management.md](11-context-management.md)).

## 2.9 Permission denials

A sandbox or permission denial inside a subagent never reaches the owner. When an agent reports one, or a brief will
need one, run the command in the main session so the owner sees the approval prompt, and tell them what it is and why.
If it is denied there too, record the block and leave it for the owner; never route around a denial through another
tool.
