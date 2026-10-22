# 10. Working with the owner

The owner decides behavior, priorities and anything irreversible. Everything else is the orchestrator's job, done with
a record the owner can review afterwards.

## 10.1 Asking questions

- **Look first** in the design, glossary, non-functional guide, backlog and decisions log. Never re-ask a settled
  question; reopening one (for example, proposing a logging change that a recorded decision had already ruled out) costs the
  owner's trust and time.
- **Ask with options and a recommendation**: the question in one line, two to four options with their consequence,
  one recommendation in bold with the reason. The owner usually answers instantly when it is framed that way.
- **Batch questions up front** before an unattended run, so it does not stall on them.
- **Record every answer** as a decision entry, in the owner's words where they matter, and edit the design if behavior
  changes.
- When the owner's answer needs interpretation, write "my interpretation" in the entry so it can be corrected.

## 10.2 Autonomy grants

- **Ask once, with the real options**, before a stretch of unattended work: e.g. "only mechanical batches", "this run
  plan's batches", "the whole next phase including batches with design judgment"; and "pace usage" versus "spend
  freely". Do not assume the cautious option; on the reference project the owner chose the widest grant because an idle machine overnight was
  the waste they cared about.
- **Record what the grant covered and what it did not**, at the time it is given. A verbal grant is what later looks
  like the assistant deciding on its own.
- **Small design gaps while away:** decide only if inside one batch and changing nothing users see, the security model
  or the data shape; mark the decision "for review"; list all such decisions when the owner returns, before anything
  else.
- **Phase ends:** agree whether to wait for a go-ahead. The reference project's owner dropped the wait as arbitrary: sweep,
  report, continue.

## 10.3 Minimal chat, detail in files

During runs, chat carries only a question the owner must answer, a result they asked for, or one line of status. No
tables, recaps or lists of what is running. Status, decisions, findings and next steps go into the session log, the
ledger and the progress logs, committed, which is where the owner reviews them. Name batches in plain words. See
[11-context-management.md](11-context-management.md).

## 10.4 Checkpoints and reports

At each phase end (or when asked), a short report: batches merged (plain words), decisions taken and those flagged for
review, open questions, known risks, what is next. A browser pass over shipped screens at phase ends is worth keeping:
the reference project's owner valued it because it finds things other tests can't.

For reports meant for others (a summary of a measurement campaign, a recommendation), put the limits or choices the
owner must set at the top, each with a recommendation.

## 10.5 Hard stops

Regardless of any autonomy grant, these wait for the owner:

- force pushes, history rewrites, deleting repositories or shared branches, changing repository or harness settings;
- anything touching production or production credentials (nothing in the build should);
- a design gap that changes behavior, beyond the small-gap rule above;
- a fix with no clear solution (cause not understood, a failed fix with no new direction);
- a gate failure that reproduces and needs a behavior decision;
- a permission or classifier denial: hand the command to the owner, never route around it.

The owner's other personal approvals (signing in to a demo, running a command that needs their session) are theirs
to do; give them the exact command to run.

## 10.6 Multiple workstations

A second machine roughly doubles test capacity. What worked on the reference project:

- **Roles:** the main session reviews, adjudicates and merges; the second machine implements test-heavy batches, runs
  gates and long measurements, so the main machine stays free for its own gate.
- **Coordination through git:** a self-contained brief, a progress log and a results folder in the workspace, pushed;
  short messages only to say "start" or "ready". Delivery is unconfirmed until its log or branch appears.
- **Trust, but check the SHA:** the second machine's gates log does not reach the first, so its gate line is accepted
  as relayed, with the SHA checked against the pushed branch.
- **One gate per machine still applies**; send each machine a new, self-contained job rather than waking a
  long-context session.
- **Machines differ:** a failure on one machine that the other never shows is a machine difference until proved
  otherwise; verify which process hit which ceiling before prescribing a change for a machine you have not inspected.

## 10.7 Usage and the owner's budget

Ask the owner how usage limits should shape the run (short window vs weekly). On the reference project: above about 80 % of the
short window, one agent at a time; the weekly limit was not a concern. The orchestrator usually cannot read usage
itself; use the owner's latest reading.

## 10.8 Writing for the owner

Plain, direct prose; the owner's preferred spelling (state it in every content brief); no unexplained ids; the decision they must make at the top.
