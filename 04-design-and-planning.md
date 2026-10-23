# 4. Design and planning

The design says what to build and is authoritative. The plan says in what order and how each piece is accepted. Both
live in the workspace, and only the orchestrator edits them.

## Before the design: start from a demo

A design written from a blank page guesses at what people need. A working demo lets them react to something real, and
reactions are cheaper and more reliable than requirements interviews.

1. **Brainstorm** with the owner and a few intended users: the problem, who has it, what a good day looks like.
2. **Write a demo spec**: one or two pages naming the screens, the story the demo tells and the synthetic data it
   needs. Leave out non-functional rules, integrations and edge cases.
3. **Build the demo fast**, with stand-ins for every outside system. It is allowed to be throwaway code.
4. **Iterate on the demo** with the people who will use it. Record each piece of feedback in the owner's words, then
   what changed. Stop when the feedback becomes refinement rather than surprise.
5. **Write the initial design from the demo.** The demo is the baseline for screens and behavior. The feedback log
   explains why it looks the way it does. Then add what a demo skips: data ownership, security, non-functional
   requirements, integrations.

Keep the demo running after the build starts. Showing each phase's real build to the same people keeps the design
honest; feedback on what users see is a good first priority group (§4.7).

## 4.1 The design folder

```
design/
  README.md                  what the product is, reading order, document index, how documents are organized,
                             how the design changes
  architecture-overview.md   the system, components, modules, data architecture, identity and access, screens
  glossary.md                the vocabulary every other document uses
  non-functional-guide.md    engineering standards with stable requirement ids
  <area or module>.md        one document per module or area, in a fixed internal order
  views/<screen>.md          one document per screen, plus shared UI conventions
  backlog.md                 work deliberately deferred, with the design hooks already in place
```

- **Reading order in the README.** Every agent reads the README, the glossary and the non-functional guide, then only
  the sections its brief names.
- **Stable requirement ids** in the non-functional guide (the reference implementation used prefixes such as `ARC-20`, `SEC-03`, `DAT-07`,
  `PERF-02`). Briefs, reviews, findings and code comments cite them, so a rule is referenced once and never
  paraphrased into drift. Never renumber.
- **A fixed order inside each module document** (reference implementation: data and constraints → read views published to others →
  behavior → API → contracts → authorization and threat model → non-functional specifics). Agents learn where to look.
- **Ownership rules in the design**, not only in code: in the reference implementation a module owns exactly one schema and is its only writer;
  others read it only through published views or contracts. A change to a contract names every consumer it affects.
- **Glossary first.** Disagreements about words become disagreements about behavior. Define each term once.

## 4.2 From draft to design

A **major feature** (new module, new integration, anything with several open decisions) starts as a **design draft** in
`plan/` (e.g. `plan/<feature>-design-draft.md`), marked "draft, not design: nothing here is authoritative until the
decisions in §1 are answered and the orchestrator edits `design/`". Shape that worked on the reference project:

1. **Status line and one-paragraph summary.**
2. **Decisions for the owner**, most important first: a table of decision, options and one recommendation (bold), with
   the reason. "Accepting the draft as a whole accepts all recommendations."
3. **Design text this feature changes or contradicts**, by document and section.
4. **Supporting design**: the actual behavior, data, API, security.
5. **Technology additions considered** (and rejected, with why).
6. **Stand-ins and integration points** it creates.
7. **Build order**: the batches.
8. **Risks and what is left out.**
9. **Design edits when accepted**: exactly which files change.
10. **Not found or not verified**: claims the draft could not confirm.

The owner answers §2; the orchestrator records each answer as a decision entry, edits `design/` per §9, and the draft is
superseded (rename it `*.superseded.md` or delete it once the design holds everything). Batches cite `design/`, never
the draft.

A design draft can be written by an agent; the decisions in it are the owner's.

## 4.3 The backlog

Two kinds of deferred work, kept apart:

| Where | What | Example |
|---|---|---|
| `design/backlog.md` | Deferred **scope**: features beyond the build, each with what it is, why it matters, what is decided and open, what blocks it, rough size | "Two-way integration with the ticketing system: undesigned; blocked on access" |
| Ledger §Follow-ups | Deferred **work items** found during batches: findings that can wait, deliberate shortcuts with a ceiling, edge cases | "Paging on the orders list reads all matching rows; ceiling ~X rows; pick up in the next performance batch" |

A plain-language summary of the scope backlog for the owner (the reference project kept one written for non-engineers) helps them
prioritize.

## 4.4 Major features vs regular backlog items

| | Major feature | Regular item |
|---|---|---|
| Starts as | A design draft with a decision table | A follow-up row or backlog row |
| Owner involvement | Answers the decisions before any brief | Usually none; adjudication covers it |
| Plan | Its own phase or batch group, with dependencies and parallel eligibility | A small batch, or folded into a related batch |
| Design | Edited before the first batch | Edited in the batch, if behavior changes |
| Review | One reviewer per area across its sub-batches | Normal |

Small follow-ups are cheaper grouped: a "cleanup" batch that takes ten related follow-up rows costs one brief, one
review and one gate.

## 4.5 The execution plan

`plan/execution-plan.md`:

1. **How to read a batch card.**
2. **Sequencing principles.** These held up well on the reference project: foundations first (gates, database plumbing, shared kernel,
   authorization, local stand-ins); data before the features that read it; a walking skeleton (end to end, signed in,
   seeded) early; contracts before consumers (where two modules need each other, the first ships without the link and a
   named later batch adds it); a module before its screen; no "tests later" batches.
3. **Definition of done**, applying to every batch: scope delivered and nothing else; tests by layer, including denied
   and negative cases and a violating write for every new invariant; schema only through new migrations, none edited
   after merge; every endpoint authorized; documentation current; the gate passes from a clean checkout; assumptions,
   questions and design errors reported, none resolved silently.
4. **Phases**, each ending in a checkpoint.
5. **Batch cards**:

| Field | Meaning |
|---|---|
| Id | Short code; a letter prefix per kind of work (e.g. `F` foundations, `L` local stand-ins, `P` pipeline, `A` app backend, `S` seeding, `W` web, `K` packaging, `H` hardening, `R` refactor); a letter suffix for a split (`W01a`) |
| Repo | Repositories it changes |
| Depends | Batches that must be merged first |
| Parallel | `—` for serial, or the batches it may run beside, each in its own worktree |
| Agents | Implementer / reviewer model; mark mechanical work for a runner |
| Design | Sections to read first |
| Deliver | What exists when merged |
| Accept | Checks beyond the definition of done |
| Review focus | Where the reviewer spends extra attention |

When a card is split or superseded, keep the old card marked superseded so its scope and references remain findable.

## 4.6 Run plans

For a stretch of work outside the execution plan's order (an overnight run, a cleanup push, a performance campaign),
write a **run plan** in `plan/run-plan-<name>.md`:

- Why it exists and the owner's goal, in their words.
- Sources it was built from (inventories, measurement reports).
- **A "Status — resume here" table** at the top, updated after every step with the ledger:

| # | Batch (plain words) | Repo | State | Next action | Log |
|---|---|---|---|---|---|
| 1 | **B1** · test isolation and hardening | app | merged `<sha>` (gate attempt 2 green) | — | `progress/B1-*.md` |
| 2 | **B2** · data-growth fixes | app | reviewing | adjudicate | `progress/B2-review.md` |

- Questions asked up front, with answers (so an unattended run does not stall on them).
- What the owner's authorization covered and what it did **not**.

A cold session reads the session log, then the active run plan's status table, then the ledger.

## 4.7 Priority ordering

Agree a priority order with the owner and record it; pick the next batch from the highest group with work ready.
Lower groups use only idle capacity. An example late-stage order: (1) feedback on what users see in the demo; (2) help
and documentation; (3) the next major feature's MVP; (4) everything else, ordered by the owner when reached. Re-ask when the product's
stage changes (e.g. before a handoff the owner wanted every known fixable issue fixed first).

## 4.8 Design changes are recorded in the same batch

When a batch changes documented behavior, or finds design text wrong, the orchestrator edits `design/` in that batch
and cites the batch id. The decision behind the change goes in the decisions log. A component
repository never holds its own copy of the design; it refers to sections.

*Why:* with many cold readers, a design that lags the code by even one batch misleads the next brief writer, the next
implementer and the next reviewer.

## 4.9 Questions that need outside data

When a design question can only be answered by data you cannot see (production field shapes, fill rates), collect it as
a numbered question in a dedicated file with a "why" line, for the owner to run when they can. Build against the
stand-in's shape meanwhile when nothing depends on the answer. Keep such questions aggregate-only (no names or
personal data).
