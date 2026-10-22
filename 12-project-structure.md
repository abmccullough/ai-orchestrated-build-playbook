# 12. Project structure

Two kinds of git repository: one **workspace** that the orchestrator works from (design, plan, records, agent
definitions), and one or more **component repositories** that hold the product code. Keeping them apart means product
history holds only product changes, and the build's state (ledger, logs, decisions) never ships with the product.

## 12.1 Why separate repositories

- Components that have their own release lines get their own repositories. In the reference implementation: the application, the data pipeline,
  and a GitOps environment repository (kept separate so CI never holds cluster credentials).
- The workspace is one more repository: one home for the design, the plan and the build state.
- A monorepo was considered and rejected on the reference project because it would merge release lines the design keeps apart. Its one
  advantage, a design change in the same pull request as the code, is kept by rule instead: the orchestrator edits
  `design/` in the same batch and cites the batch id.

Record the reasoning and each repository's contents in `plan/repositories.md`.

## 12.2 The workspace repository

```
<workspace>/                          its own git repo; the orchestrator session starts here
  CLAUDE.md                           orchestrator instructions (template: templates/CLAUDE.workspace.md)
  README.md                           what this repository is
  .claude/agents/                     agent definitions: implementer, reviewer, design-brief-writer, runner
  .claude/settings.json               harness settings and environment for agent shells (tech notes)
  design/                             the authoritative design (04-design-and-planning.md)
    README.md, glossary.md, non-functional-guide.md, <modules>/, views/, backlog.md
  plan/                               only the orchestrator edits this folder
    orchestrator-session.md           session log; latest HANDOFF on top
    ledger.md                         batch status, §Follow-ups, §Flake register
    decisions.md                      IMP-n decisions
    execution-plan.md                 phases, batch cards, definition of done
    run-plan-<name>.md                an active run with its "Status — resume here" table
    testing-discipline.md             the current testing rules
    performance-lessons.md            performance patterns and instances
    integration-points.md             stand-ins for environment-supplied details
    repositories.md                   the repositories, why, how they link, local layout
    prerequisites.md                  workstation tools and versions
    orchestration.md                  (optional) the detail and reasons behind CLAUDE.md
    <feature>-design-draft.md         drafts before they become design
    briefs/
      _implement.md _review.md _fix.md _triage.md _gate.md     templates
      <ID>-implement.md, <ID>-review.md, <ID>-fix-1.md, <ID>-gate.md   every issued brief, kept
  progress/                           agents write here: one log per dispatch, named after its brief
    <ID>-implement.md, <ID>-review.md, <ID>-fix-1.md, <ID>-gate.md, ...
    gates.log                         one line per gate, appended only by the gate wrapper script
  scripts/                            workspace tooling
    run-gate-locked.<ext>             the gate wrapper: machine-wide lock, runs the repo's gate, appends to gates.log
    clone-repos, check-prereqs        setup
    new-worktree, remove-worktree     parallel batch worktrees with their own database names and ports
    remove-merged-branches            cleanup (dry run by default)
    compare-seed-dumps                determinism check for the phase sweep
  repos/                              git-ignored: primary clones of each component repository, on main
    <app>/  <pipeline>/  <gitops>/
  .worktrees/                         git-ignored: one worktree per parallel batch
    <repo>--<ID>/                     on batch/<ID>
```

### Ignored in the workspace, and why

| Ignored | Why |
|---|---|
| `repos/`, `.worktrees/` | Component clones have their own history; tracking them would nest repositories |
| `*.log` (except `progress/gates.log`, which you force-add or exempt) | Tool and build noise; the gates log is evidence and is tracked |
| Local tool indexes and caches, `node_modules/` | Machine-specific and regenerated |
| Files a tool regenerates on every run (e.g. a generated `.claude/CLAUDE.md` with a timestamp) | They leave the tree permanently dirty |
| Large generated evidence (database dumps, diffs) | Kept locally as evidence; the log records the result |
| `.env`, account values, photos of source systems | Secrets and private material never go into git |

Note in the ignore file who decided each entry and when; an unexplained ignore gets removed by the next person.

## 12.3 A component repository

Generalized from the reference implementation's application repository:

```
<component>/                          its own git repo; agents work here (or in a worktree of it)
  CLAUDE.md                           agent rules for this repo; states the gate (template: templates/CLAUDE.component.md)
  README.md
  verify.<ext>                        the gate: every check from a clean checkout, lanes by diff, prints GATE-LANES
  <solution or package manifest>      locked dependency versions; toolchain version pinned
  src/
    <hosts or entry points>/          composition only, no business logic
    <shared kernel / libraries>/
    <modules>/<Module>.Contracts/     what other modules may reference
    <modules>/<Module>/               implementation; owns one schema
    <tools>/                          e.g. the synthetic data generator
  tests/
    Unit/  Integration/  Architecture/   test layers (architecture tests enforce module boundaries and bans)
  web/                                front end (own package and lockfile), its tests beside it
  e2e/                                browser end-to-end tests
  db/changelog/<module>/NNNN-<description>.sql   migrations: only new files, never edit a merged one
  openapi/                            API snapshot, diffed by the gate
  dev/                                local stand-ins: compose files, env and seed scripts, local settings
  deploy/                             images, charts, environment packaging
  pipelines/                          CI definitions
  scripts/gate/                       gate helpers (a change here runs every lane)
  docs/                               module READMEs, runbooks
  <pin file>                          e.g. pipeline.lock: the commit of another component this one consumes
```

### Ignored in a component repository, and why

Build output and test results (`bin/`, `obj/`, `dist/`, coverage, test reports); `node_modules/`; local environment
material (`.env`, `dev/.secrets/`, `dev/.certs/`, local data folders, `*.local.json`, generated files the gate exports
from a pinned commit of another repository); measurement results; editor folders except shared extension lists. The
rule: anything generated, machine-specific or secret is ignored; anything a fresh clone needs to pass the gate is
tracked or produced by a documented script.

## 12.4 How they link

- **Local layout:** primary clones and worktrees sit at the same depth under the workspace
  (`repos/<repo>`, `.worktrees/<repo>--<ID>`), so a relative path such as `../../design/` reaches the design from
  either. Component `CLAUDE.md` files refer to the design by that relative path and by section; they never copy it.
- **Pins between components:** when one component consumes another's output (in the reference implementation the app consumed the pipeline's
  database changelog, stub and data generator), it records the exact commit in a tracked pin file (`pipeline.lock`).
  Tests and seeds take the other component from that commit, found through an environment variable defaulting to the
  workspace clone. Bump the pin after every merge that changes what is consumed; a bump to identical code needs no gate.
- **Shared contracts** across languages are proved by test vectors published in one repository and asserted in the
  other.
- **Design** lives only in the workspace. **Gates** are run through the workspace's wrapper and recorded in the
  workspace's `gates.log`, whichever repository they test.
- `plan/repositories.md` holds all of this for your project: why these repositories, each one's contents table, how they
  link, the local layout.

## 12.5 Who may write where

| Who | May write | Never |
|---|---|---|
| Orchestrator | `plan/`, `design/`, `CLAUDE.md`, `.claude/agents/`; merges and pushes `main` of component repositories after a green gate | Product code or tests |
| Implementer | Its brief's checkout (a component repository's primary clone or its worktree), on its batch branch; its own progress log in `progress/` | `plan/`, `design/`, `main`, any other checkout |
| Reviewer | Its progress log; temporary probes in its own scratch clone, restored. On a fix brief, the named checkout and branch | Anything else on a review or triage brief |
| Design-brief writer | `plan/briefs/<ID>-implement.md` and its progress log | Ledger, decisions, run plans, `design/`, component repositories |
| Runner | Its progress log; scratch clones; commits only as its brief says | `progress/gates.log` (only the script writes it), `main` unless told |
| Gate wrapper script | `progress/gates.log`, its output files in a scratch folder | — |

Scratch clones (fresh-clone gates and reviews) live outside both the workspace and `repos/` (e.g. `C:\work\<project>-tmp\…`) so
they never appear in any repository's status. Workspace commits are always **by path**, because agents' uncommitted
progress logs sit in the same tree.
