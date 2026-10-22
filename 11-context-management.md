# 11. Context management

How the orchestrator manages its own context and the owner's usage. The orchestrator's context is the scarcest
resource in the system: it holds the plan, the open decisions and every batch in flight, and every token in it is paid
for again on every turn. Keep it lean, and when it grows, hand off to a fresh session rather than dragging it along.

## 11.1 The practice: hand off to a fresh session

Earlier notes on the reference project said "compact the conversation while waiting on agents". In practice, **handoffs to fresh sessions**
worked better and became the norm: a fresh session that reads a written handoff costs far less per turn than a long
session, and a summary written on purpose by the orchestrator is more reliable than an automatic compaction.

**When:** the session has grown large (many batches merged, long reports read), at a natural break (a batch merged,
nothing waiting on a decision), before a long unattended run, or when the owner asks.

**How:**

1. **Write a HANDOFF block at the top of the session log** (`plan/orchestrator-session.md`), headed with the clock
   time and "Read this first; it supersedes everything below." It contains:
   - **Standing rules in force** for this stretch: agent limit, autonomy grant, one gate per machine, chat style, any
     temporary rule from the owner (with who said it and when).
   - **Running**: each agent or remote session with its batch (in plain words), model, checkout and branch, brief
     path, progress log path, and its last known state.
   - **Owed, in order**: what the orchestrator must do next, numbered, each with enough detail to act cold (paths,
     SHAs, commands, the decision still needed).
   - **State, checked against git**: each repository's `main` SHA, cross-repository pins, checkouts clean and on
     `main`, leftover worktrees, anything on another workstation.
   - **Decisions and lessons from this session** not yet elsewhere (most should already be in the decisions log,
     ledger, testing or performance files; the handoff points to them).
2. Update the ledger and run-plan status table, if not already current.
3. **Commit the workspace by path and push.**
4. Start a fresh session (`/clear` in Claude Code) with the prompt "read the top of the session log" (the orchestrator
   instructions already say to read it first, then the ledger).
5. The new session checks the handoff against git and the progress logs (what an agent believed when it last wrote may
   be stale), writes a short CHECK note above the handoff with anything that changed, and continues.

Each new handoff goes above the previous one and marks it superseded. Older ones stay as history.

Reference implementation example (abridged, identifiers made generic):

```markdown
## HANDOFF — <date> 21:02 (clock read). Read this first; it supersedes everything below.

**Normal agent rules** (the owner, 14:19: up to 3 agents, no per-agent go-ahead). One gate per machine.
Chat brief; name batches in plain words.

### Running
- **B21 implementing** (report read performance fixes F-1..F-4); implementer-opus, serial `repos/app` on
  `batch/B21` from `a1b2c3d4`; brief `plan/briefs/B21-implement.md`, log `progress/B21-implement.md`.
- **Workstation 2** runs the scheduled-report load test; brief `plan/briefs/B22m-W2.md`, log
  `progress/B22m-measure.md`; watch for "PHASE C READY" there after `git pull`.

### Owed
1. Phase C measurement when W2 says ready: commands at the end of `progress/B20m-measure-W1.md`.
2. Then fix F-1 and F-2 (scaling costs, fix now): evidence in the final entry of `progress/B20m-measure-W2.md`.
3. Report to the owner when W2 is done: limits for the owner to set; pool size is the owner's call.

### State (checked against git, 21:02)
- App `main` `a1b2c3d4`; pipeline `main` `e5f6a7b`; app pin `c8d9e0f` (ancestor, by design).
- Both primary checkouts on `main`, clean. Old worktrees B15, B17, …: leftovers; check each against the ledger
  before removing.
```

## 11.2 When compaction is still fine

A short idle wait inside one task (one batch's reviewer is running and nothing else is pending) can still end with a
one-line status and a note that it is a good point to compact. Do not fill such a wait with speculative reading for
work whose inputs depend on the pending report. For anything longer or spanning several batches, write a handoff.

## 11.3 Keep the orchestrator's context lean

- **Delegate sweeps.** A question that means reading across many files (an inventory of open follow-ups, "where is X
  used", a backlog review) goes to an agent; keep only its conclusion. A single known lookup is done directly: an
  agent for one read costs a cold start.
- **Read progress logs, not transcripts**, to know where an agent is. The log's Status block is designed for that.
- **Read big files in portions or by search.** A ledger or decisions log grows to hundreds of kilobytes; read its
  header, the row you need, or grep for the id. Never load it whole.
- **Briefs and reports point to files.** A design-brief writer keeps brief-writing out of the orchestrator's context;
  its report is short (path, scope in a line, questions, contradictions) and does not paste the brief.
- **Don't re-read what you just wrote**, and don't re-derive a rule that already lives in the testing or performance
  file: cite it.

## 11.4 Agents start cold, on purpose

Every agent starts from a self-contained brief and its progress log, so none needs the orchestrator's history. That is
also what makes resuming cheap: **resume the work, not the agent**. After a pause, a usage stop or a new session,
dispatch a fresh agent with the same brief and "resume from the progress log". Waking an old agent reloads and
re-caches its whole context, which costs more than a cold start. The one exception is a warm fix within the prompt
cache lifetime after an ordinary run ([02 §2.5](02-roles-and-agents.md)).

The same applies to other workstations: send them a new, self-contained job; don't wake a long-context session to
continue.

## 11.5 Minimal chat, detail in files

During runs, chat carries only a question the owner must answer, a result they asked for, or one line of status.
Status, decisions, findings and next steps go into the session log, the ledger and the progress logs, which is where the
owner reviews them and where a fresh session finds them. A long chat recap costs context now and is lost at the next
handoff. ([10 §10.3](10-working-with-the-owner.md).)

## 11.6 Usage

- Above about 80 % of the short-window usage, drop to one running agent so nothing stalls mid-batch for hours; raise it
  again after the reset. The orchestrator usually cannot read usage itself: use the owner's latest reading.
- Ask the owner whether the longer-window (weekly) limit matters; on the reference project the owner said not to throttle
  for it.
- Set the subagent prompt-cache lifetime long enough to cover a gate's wait (the reference project used one hour); a short default turned
  long gates into full cache re-writes. Harness details are in the tech notes in [`tech/`](tech/).
