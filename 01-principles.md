# 1. Principles

The standing preferences that shape every other rule. Each has a short reason and, where useful, where it came from in
the reference implementation. When a detailed rule elsewhere would contradict one of these, the principle wins and the record says why.

## 1.1 The design is authoritative

Code that disagrees with the design is a defect, unless the design is changed on purpose. The design is changed only by
the orchestrator, in the same batch as the code that needs it, and the change is recorded.

*Why:* agents start cold and each one reads the design. If code is allowed to drift, every later agent reads a design
that no longer describes the system, and the drift compounds. One authority keeps many cold readers consistent.

## 1.2 Raise gaps; never assume

When the design is silent, ambiguous or contradictory on something that changes behavior, nobody guesses. Agents build
everything that does not depend on the answer and report the question. The orchestrator looks first in the design,
glossary, non-functional guide, backlog and decisions log; if the answer is not there, it asks the owner with options
and a recommendation. The answer becomes a decision entry, and a design edit when behavior changes.

Behavior-neutral choices (private names, file layout) are the agent's to make and list as assumptions.

*Why:* a guessed behavior looks finished and gets built on. Finding it later costs a fix, a re-review and every
dependent batch. A question costs one round trip.

**While the owner is away** (reference implementation rule, agreed with the owner): the orchestrator may decide a small gap itself if it
stays inside one batch and changes none of what users see, the security model or the data shape. It records it as a
decision "decided while the owner was away; for review", and lists all such entries when the owner returns. Bigger gaps
block only their own batch.

## 1.3 Humble stand-ins

Where the real environment will supply something you do not have (enterprise secrets layout, identity provider
values, certificate bundles, real source field names), build the smallest stand-in that exercises the shipped code and
record the need in an **integration-points register**. Do not invent a production design, and do not add local-only
hardening that has no production counterpart. Keep the hardening the shipped code must enforce by design. Do not undo
earlier choices unless they get in the way.

*Why:* on the reference project the owner judged that strong local security choices create hard-to-reverse,
hard-to-find problems when moving to the real environment. A register of needs transfers cleanly; an invented design
has to be discovered and unpicked.

## 1.4 The laziest thing that works

Prefer, in order: no code; what the repository already has; the standard library or a native platform feature; an
already-installed dependency; the shortest thing that works. No interface with one implementation, no configuration
for a value that never changes, no scaffolding for a later batch, no column nothing reads.

It never applies to validation at a trust boundary, error handling that prevents data loss, security controls,
accessibility, or anything the design or owner asks for by name.

A deliberate shortcut with a real ceiling is recorded as a **follow-up row in the ledger** (the ceiling, the upgrade
path, the batch that must pick it up), not as a code comment, so there is one deferral register.

*Why:* on the reference project an early batch shipped four speculative columns that nothing read and that an existing
catch-all field already carried. Speculative code costs review time, test time and future confusion.

## 1.5 Trust but record

Give agents and the orchestrator real autonomy, and keep a full audit trail: every decision in the decisions log with
its reason, every batch in the ledger, every dispatch with a progress log, every gate with a recorded line. Integrity
rules stay enforced regardless of autonomy.

*Why:* the owner cannot watch everything. A complete record lets them review afterwards, lets a cold session resume,
and turns "the assistant decided on its own" into "the assistant decided under this grant, here is why".

## 1.6 Every test run serves a purpose

Before any test run, gate, re-review or seed check, name what it could catch that nothing already recorded covers, for
this change. If the answer is nothing, do not run it; record why it was skipped. Size the run to the size and risk of
the change, not to the habit of the step.

This outranks every other testing rule. *Why:* on the reference project a full 35-minute browser gate ran on a
version-pin bump to identical code that had already been gated; it could catch nothing. Repeated gates were the single
largest waste of the build. Details: [08-testing-discipline.md](08-testing-discipline.md).

## 1.7 A test must be able to fail

A test for an acceptance item, a design rule or a fixed finding counts only after it has been seen failing for the
right reason against deliberately broken code, then passing. A test that cannot fail is deleted, not kept "for defense
in depth".

*Why:* a test that always passes guards nothing and gives false confidence to every later reviewer.

## 1.8 A report is not evidence

An agent's report is a claim. A gate counts only by its recorded line in the gates log. A value, default or literal a
report describes is read from the commit before anyone relies on it. A mechanical agent reports what it ran and saw,
never a cause.

*Why:* on the reference project an orchestrator raised two "defects" from a report's description of constants that were not in the
commit, and told the owner they were real; one `grep` would have caught it. A runner once reported a passing gate with
invented counts while the gate was still running.

## 1.9 Fix scaling costs proactively

Build for 100x to 1000x the local data. A cost that grows with data and has a known fix (a missing index, a
per-row query, a whole-set read, aggregation in application code instead of the database, a row per date) is fixed
when seen, even when today's numbers are well inside target. "Watch" is only for costs with no growth term.

*Why:* local mid-scale margins say little about production scale, and the fixes are cheapest before other code copies
the pattern. Details: [09-performance-and-scale.md](09-performance-and-scale.md).

## 1.10 Edge cases to the backlog

Simplest thing that works. A rare interleaving, a cosmetic edge, an unlikely data shape or a test-only nit goes to a
follow-up row, to be worked once the product is feature complete, not into a fix loop.

Fix now only what has a reason (see [03-batch-lifecycle.md](03-batch-lifecycle.md) §6): breaks the gate, security or
data integrity, users see it, likely in normal use, a later batch builds on it, contradicts design others rely on, or a
scaling cost with a known fix.

**Before a handoff** the weighting flips: everything known and fixable is fixed rather than handed over as someone
else's surprise; edge cases still go last, but they are worked.

## 1.11 Integrity over visibility

Spend security effort on protecting the system: writes, resource exhaustion, the audit trail, startup and configuration
integrity. Visibility rules (who sees which rows or columns) get more leeway: when making a visibility rule stricter
would add code, tests or a dispatch for an edge case, list it with a recommendation and let the owner decide instead
of defaulting to "more restrictive".

On the reference project the owner also ruled that findings revealing only counts, totals or timing outside a person's access are
accepted, not fixed; anything revealing a hidden row's contents stays a defect. Whether that holds for your domain is
the owner's call; ask early.

## 1.12 Implement small, review by area

Split a batch that spans areas or would fill an implementer's context into sub-batches, each a fresh implementer
dispatch, in sequence on one branch. One reviewer covers every sub-batch in the same area, because it needs the same
context. The group shares one merge gate.

*Why:* implementer context bloat lowers quality; a reviewer paying to load the same design and code three times is
waste.

## 1.13 No fixed fix-loop count

A clear, material fix is always made, whatever the loop number. A batch goes to the owner only when there is no clear
solution: the cause is not understood, a fix failed and the next attempt would be another guess, or the fix needs a
behavior decision. *Why:* the reference project first used a two-loop stop; twice it brought the owner a clear fix that
they always approved. The spirit is "don't thrash without direction".

## 1.14 Name things in plain words

Every batch id, decision id or finding id shown to the owner carries a few plain words beside it the first time in a
message ("B12b (the remaining 11 sample email templates)"). *Why:* the owner does not memorize ids.

## 1.15 Read the clock

Never state a time or an elapsed time from feel. Record the clock when dispatching; read it again and subtract when
checking. *Why:* on the reference project guesses ran up to 40 minutes off, triggered needless status requests and put wrong times into
the resume notes a cold session trusts.
