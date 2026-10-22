# <component> — Agent Rules

<!-- Copy to the root of each component repository as CLAUDE.md. Replace every <...>. Keep it to the rules an agent
needs in this repository; the process lives in the workspace's CLAUDE.md. -->

<One line: what this repository is.> Work arrives as batch briefs from the orchestrator (`<workspace>/CLAUDE.md`);
follow the brief, and these rules.

## Rules

- **Design:** `../../design/` from this checkout or a worktree. Read the sections your brief cites. Never edit it, here
  or there; report design problems in your report instead. Never edit the workspace's `plan/`.
- **Where you work:** only the checkout and branch your brief names. Never push `main`, never merge, never rebase a
  shared branch. Push your batch branch after each commit.
- **Structure** (<requirement ids>): <e.g. each module is `src/Modules/<M>/<M>.Contracts` plus `<M>`; a module references
  another only through its contracts; hosts hold no business logic; each module owns one schema and is its only
  writer.>
- **Database** (<ids>): every schema change is a new migration in `<path>/<module>/NNNN-<description>.<ext>`, included
  from the root. Never edit a merged migration. No other way of changing schema.
- **Version floor:** all SQL (or other platform-bound code) runs on <minimum version> and uses none of the features
  <id> excludes. Tests run on <tested version>; a newer local install proves nothing.
- **Tests ship with the code**, by layer:

| Layer | Where | What it covers | Runtime |
|---|---|---|---|
| Unit | `tests/Unit` | Logic, gate scans | seconds |
| Architecture | `tests/Architecture` | Module boundaries, banned calls (e.g. wall clock, random ids in generators) | seconds |
| Integration | `tests/Integration` | Persistence, grants, invariants (a violating write per invariant), HTTP endpoints allowed and denied | minutes; filter by module |
| Front end | `web/` | Components and screens | minutes |
| End-to-end | `e2e/` | User journeys on a live, seeded stack | the longest; merge gate only unless the brief says otherwise |

- **Fresh clone or worktree prerequisites:** <e.g. export a pinned changelog from the other component before the first
  integration test; generate local certificates before starting a stack>. Without them, tests fail in setup and the
  failure says nothing about your batch.
- **Pins:** `<pin file>` names the commit of `<other component>` this repository consumes, found through
  `<ENV_VAR>` (default `../../repos/<other>`). Never edit the pin unless your brief says so.
- **Synthetic data:** any scenario a behavior, rule or screen needs is added to `<generator path>` in the same batch,
  with exact expectations.
- **Seeded database:** reseed (`<seed command>`) before tests that read it.
- **Local stand-ins** are configuration only: no environment branches in code, no production endpoints, no secrets.
- **Containers:** start only what you need; stop what you started; never stop or remove anything you did not start.
  One live end-to-end stack per machine.

## Gate

`<gate command>` (e.g. `pwsh ./verify.ps1`). Run through the workspace's wrapper,
`<workspace>/scripts/run-gate-locked.<ext> -Checkout <path>`, which records the result in `progress/gates.log`; that
line is the only evidence a gate ran. Implementers run targeted tests, not the gate, unless the brief says otherwise
(`<workspace>/plan/testing-discipline.md`).

Steps, in order:

1. Toolchain version within the pinned band.
2. Container engine available (checked early so later steps fail fast).
3. Locked dependency restore (fails on a stale lockfile).
4. Build with warnings as errors.
5. Migration validation and **immutability** (every migration on `origin/main` byte-identical on the branch).
6. Static scans (<e.g. SQL feature floor, extension allowlist>).
7. Test projects: unit, integration, architecture, with coverage; fail under <N> % line coverage.
8. API snapshot diff.
9. Front end: install, lint (including module-boundary rules), typecheck, unit tests with coverage, build, build
   checks (hashed assets, no inline scripts, no external URLs).

`<gate command> -E2E` adds: end-to-end dependencies, local certificates, a reseed, the stack up and answering a real
request, the browser tests, and teardown (pass, fail or interrupted). It refuses to start while another stack is up.

### Lanes

Every step belongs to a lane (<e.g. backend, web, e2e>). By default the gate computes from
`git diff --name-only $(git merge-base origin/main HEAD)...HEAD` which lanes the change needs, lists skipped steps as
`SKIP`, and prints its last line as `GATE-LANES: <lanes>[ (full: <reason>)]`.

| Lane | Runs when the diff contains |
|---|---|
| <backend> | <any path outside web/, e2e/ and *.md> |
| <web> | <web/, e2e/, or the API snapshot the client is generated from> |

**Any uncertainty runs every lane:** no merge base, a failed git command, an empty diff, a dirty tree, a change to the
gate's own machinery, or `-Full`. Narrowed runs mark themselves: `| SINCE: <sha>` (classified against an earlier gated
commit) and `| ONLY` (explicit lanes); neither is a full gate.
