# Orchestrator session — plan and progress log

**Read the top of this file first after a cold start** (then `plan/ledger.md`). The newest HANDOFF is at the top and
supersedes everything below it. The orchestrator updates this file after every step, together with the ledger, and
writes a new HANDOFF before moving to a fresh session.

## CHECK — <date time> (clock read; new session)
- <what the new session found when it checked the handoff against git and the progress logs; anything that changed>

## HANDOFF — <date time> (clock read). Read this first; it supersedes everything below.

**Standing rules in force:** <agent limit; autonomy grant and what it does not cover; one gate per machine; chat style;
temporary rules from the owner, with when they said them>

### Running
- **<BATCH-ID> (<plain words>) <implementing | reviewing | gating>** on <machine>: <agent type/model>, <checkout>,
  `batch/<id>` from `<sha>`; brief `plan/briefs/<...>.md`, log `progress/<...>.md`. Last known: <state>. Dispatched
  <time, clock read>.
- **<Other machine>:** <job, brief, log; how it reports ("watch for X in its log after git pull")>

### Owed, in order
1. <next action, with paths, SHAs, commands, and the decision still needed>
2. <...>

### State (checked against git, <time>)
- <repo> `main` `<sha>`; <repo> `main` `<sha>`; pins <...>.
- Primary checkouts on `main`, clean. Worktrees: <list, in use or leftover>.
- Open questions with the owner: <...>. Decisions flagged for review: <IMP-n ...>.

### Decisions and lessons this session
- <pointers to IMP entries, testing-discipline or performance-lessons edits>

---

## Session log <date>
- <time (clock read)> <one line per step>
