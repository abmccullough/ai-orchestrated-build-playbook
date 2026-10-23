# AI-orchestrated build playbook

A repeatable process for building a real product with one AI **orchestrator** session that plans and adjudicates, and
short-lived **subagents** that write, test and review the code, under a human **product owner** who decides behavior.

It is distilled from a real multi-repository build (the reference implementation), run with Claude Code on Opus as the
orchestrator. Everything here worked there; the reasons are given so you can judge when a rule does not fit your
project. Details from that project appear only as labeled examples.

> **What this is for.** A basic workflow for real work with a human in the loop, not a recipe for unattended
> generation. The owner decides behavior, sets priorities and approves anything irreversible. The agents do the
> building and checking, and every step leaves a record the owner can audit. Treat it as a starting point: keep the
> parts that pay for themselves on your project and drop the rest.
>
> **Start from a demo.** Before writing the design, go from brainstorming to a short demo spec, build a working demo
> from it, and iterate on the demo with the people who will use it. The demo you end up with, plus what you learned
> changing it, is the baseline for the initial design
> ([04-design-and-planning.md](04-design-and-planning.md#before-the-design-start-from-a-demo)).

- **Technology-specific notes** (stack, database, containers, secrets, browser tests, OS shell, cloud, Claude Code
  harness quirks) are in the tech notes in [`tech/`](tech/).
- **Lessons learned and things not to do** are in [`addendum-lessons-learned.md`](addendum-lessons-learned.md).

## The shape in one paragraph

The design is written first and is authoritative. An execution plan cuts it into **batches**, each a small,
reviewable unit of work with dependencies and acceptance. For each batch the orchestrator issues a self-contained
**brief**; an implementer agent builds it on its own branch and reports; an independent reviewer agent checks it and
reports findings; the orchestrator rules on every finding; a fix loop runs if needed; a cheap **runner** agent runs the
full gate on the exact commit to be merged, from a fresh clone; the orchestrator merges and records. Every agent keeps
a progress log so any of them can be resumed cold. The owner answers behavior questions, sets priorities and approves
anything irreversible.

## Day one: starting a new project

1. **Create the workspace repository** (one per project, separate from the product repositories). Copy in this
   playbook's `templates/`:
   - `templates/CLAUDE.workspace.md` → the workspace's `CLAUDE.md` (or your agent harness's equivalent).
   - `templates/agents/*.md` → `.claude/agents/`.
   - `templates/briefs/*.md` → `plan/briefs/` (keep the leading underscore: `_implement.md`, …).
   - `templates/ledger.md`, `decisions.md`, `testing-discipline.md`, `orchestrator-session.md`, `run-plan.md` →
     `plan/`.
   - Create empty `design/`, `progress/`, `progress/gates.log`, `plan/performance-lessons.md` and
     `plan/integration-points.md`.
2. **Fill the placeholders** (`<...>`) in the orchestrator instructions: repositories, paths, gate commands, agent
   limit, models.
3. **Create the product repositories**, clone them under the workspace (`repos/<name>`, git-ignored), and give each a
   `CLAUDE.md` from `templates/CLAUDE.component.md` stating its gate command and its own rules for agents. Layout:
   [12-project-structure.md](12-project-structure.md).
4. **Write the gate.** One command per repository that builds and runs every check from a clean checkout, plus a
   wrapper script that runs it under a machine-wide lock and appends one line to `progress/gates.log` (time, exit
   code, duration, `clean`/`DIRTY`, commit SHA, output file, lanes). That line is the only evidence a gate ran
   ([08-testing-discipline.md](08-testing-discipline.md)).
5. **Write the design** (or move it in), starting from the demo baseline
   ([04-design-and-planning.md](04-design-and-planning.md#before-the-design-start-from-a-demo)): `design/README.md` with a reading order, a glossary, a non-functional guide
   with stable requirement ids, one document per module or area, and `design/backlog.md` for deferred scope
   ([04-design-and-planning.md](04-design-and-planning.md)).
6. **Write the execution plan**: phases, batch cards with id, repository, dependencies, parallel eligibility, design
   sections, deliverables, acceptance and review focus, plus a definition of done that applies to every batch.
7. **Seed the ledger** with one `pending` row per batch.
8. **Agree the standing preferences with the owner** ([01-principles.md](01-principles.md)): how much autonomy, the
   agent limit, which decisions always come back to them, the hard stops. Record each answer as a decision entry.
9. **Run Phase 0**: prerequisites check, repositories, an empty gate passing on `main` in each repository.
10. **Start the loop** at the first batch whose dependencies are merged ([03-batch-lifecycle.md](03-batch-lifecycle.md)).

On every later session start: read the top of the session log (the latest handoff), then the ledger; check each checkout is clean and on `main`;
resume in-flight work from its progress log ([06-records.md](06-records.md)).

## Files

| File | What it covers |
|---|---|
| [01-principles.md](01-principles.md) | The standing preferences and why each exists |
| [02-roles-and-agents.md](02-roles-and-agents.md) | Orchestrator and each agent: duties, limits, models, cold starts, concurrency |
| [03-batch-lifecycle.md](03-batch-lifecycle.md) | One batch from design to merge, step by step, with ledger statuses |
| [04-design-and-planning.md](04-design-and-planning.md) | The design folder, drafts, backlog, execution plan, run plans, priorities |
| [05-briefs-and-reports.md](05-briefs-and-reports.md) | Writing briefs; implementer and reviewer reports; reading them |
| [06-records.md](06-records.md) | Ledger, decisions, testing and performance files, session log, progress logs, gates log, batch ids |
| [07-branches-worktrees-merging.md](07-branches-worktrees-merging.md) | Branching, worktrees, the merge sequence, combined gates |
| [08-testing-discipline.md](08-testing-discipline.md) | Who runs what, fail checks, gates, triage, flakes, end-to-end scope |
| [09-performance-and-scale.md](09-performance-and-scale.md) | Building for 100x–1000x, recurring query patterns, measurement |
| [10-working-with-the-owner.md](10-working-with-the-owner.md) | Questions, autonomy grants, chat vs files, checkpoints, hard stops |
| [11-context-management.md](11-context-management.md) | Keeping the orchestrator's context lean; handoffs to fresh sessions; usage |
| [12-project-structure.md](12-project-structure.md) | Workspace vs component repositories: annotated trees, what is ignored, how they link, who writes where |
| [templates/](templates/) | Stack-neutral orchestrator instructions, agent definitions, briefs and record skeletons |
| [addendum-lessons-learned.md](addendum-lessons-learned.md) | Mistakes made and how to avoid them (separate document) |
| [tech/](tech/) | Technology-specific notes (separate documents) |

## What to adapt first

- **The agent limit and models** depend on your machine and subscription; the reference implementation settled on a few concurrent
  agents on a typical developer workstation, Opus for implement and review, a small model for runners.
- **Gate length** drives most of the testing rules. The reference implementation's full gate with browser tests took
  well over a quarter of an hour, which is why there is exactly one per merge. With a gate of a few minutes you can afford more.
- **The owner's availability.** On the reference project the owner was away for long stretches; the autonomy rules in
  [10-working-with-the-owner.md](10-working-with-the-owner.md) exist for that.

## License

[MIT](LICENSE)
