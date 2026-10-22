# 8. Testing discipline

Generalized from the reference project's testing-discipline file, which became the single current testing rule after decisions started
getting lost in the decisions log. Keep your own copy (skeleton: [`templates/testing-discipline.md`](templates/testing-discipline.md))
and edit it in the same commit as every testing decision or lesson.

Tool-specific mechanics (filters, container quirks, browser locators, build caches) are in the tech notes in
[`tech/`](tech/).

## 8.1 Principles

0. **Every test run serves a purpose, proportional to the risk of the change.** Before any run, name what it could
   catch that nothing recorded already covers. If nothing, skip it and record why. This rule outranks the rest.
1. **A test must be able to fail.** It counts only after being seen failing for the right reason against broken code,
   then passing. A test that cannot fail is deleted.
2. **A report is not evidence.** A gate counts only as its gates-log line. A value in a report is read from the commit
   before anyone relies on it. A runner reports observations, never a cause.
3. **A full run belongs where two lines of work combine**, which is the merge. Everything before it is focused.
4. **A gate is never re-run to get past a failure.** The failing tests are re-run alone and the result triaged.
5. **A partial gate is not a gate.** A gate that stopped early proves nothing about the steps it never reached.

*Why principle 3:* on the reference project, before this rule, one workstation could run dozens of gates a day
(several gate-hours) and a large share failed on faults unrelated to the batch; the same commit was routinely gated three or four times. Reviewer gate
re-runs existed because agents had once reported gates they never ran; once the gate wrapper recorded every gate
itself, that reason was gone.

## 8.2 Who runs what

| Role | Runs | Does not run |
|---|---|---|
| **Implementer** | Build with warnings as errors; the **whole** fast suites (unit, architecture: seconds); integration tests **filtered** to the modules touched and the modules consuming their contracts; front-end lint, typecheck and unit tests if it changed the front end; its fail checks. Every command and summary line in the report | The full gate or end-to-end, **unless it judges one necessary and says why** |
| **Reviewer** | From a **fresh clone**: build once (proves the pushed tree is complete) and the whole fast suites; tests the implementer did **not** run that the change could break; any module whose behavior it doubts; fail checks only where the implementer's is missing, misses its target, or a test looks weak. Each run states what it could catch | Any gate; the implementer's filtered set again; a fail check the implementer already shows |
| **Fixer** | The tests for each finding with fail checks, the touched module's tests, the fast suites | Any gate |
| **Re-reviewer** (only when required) | Only the fixed findings and their fail checks | Any gate |
| **Merge runner** | The **one full gate per batch**, on the merge commit, from a fresh clone, with end-to-end when §8.5 says so | Diagnosis |
| **Phase sweep runner** | Every repository's full gate from fresh clones of `main`, with end-to-end; determinism checks; slow sign-off tests | Diagnosis |
| **Triage agent** (reviewer on a triage brief) | Reproduces a failure with targeted runs, finds the root cause, reports cause, evidence and the smallest fix | Any edit or fix |

*Why reviewers repeat nothing without a reason:* the reference project's owner called re-running implementer tests "a big time
waster"; a repeat that can only reproduce the same counts adds nothing. A fail check is different: it re-runs a test by
nature, which is why it qualifies.

## 8.3 Fail checks

For each test covering an acceptance item, a design rule or a fixed finding:

1. Break the code the test guards (the smallest change that removes the behavior).
2. Run the test; see it fail **for the right reason** (quote the assertion output).
3. Restore the code (from version control, not a file copy, so build caches see the change).
4. Run again; see it pass. Confirm the tree is clean.
5. Record all of it in the progress log: the line broken, the failing output, the restore.

Batch the probes into one scripted pass (break, run, restore, next) rather than a turn apiece. A probe that asserts an
**absence** ("no rows left") must itself be fail-checked: disable the thing, see the count come back, restore.

The reviewer **reads** these and judges whether each break hits the behavior the test claims; it does not repeat them.

## 8.4 The gate

- One command per repository that runs every check from a clean checkout, run through a **wrapper** that takes a
  machine-wide lock and appends one line to `progress/gates.log` (time, exit, duration, `clean`/`DIRTY`, full SHA,
  output file, lanes, sleep time).
- **Lanes by diff:** the gate may skip lanes the diff cannot affect (back end, front end, end-to-end, pipeline). A change
  to the gate machinery itself runs every lane. Narrowed runs mark themselves in the lanes field so nobody mistakes
  them for a full gate.
- **The tree must not move while a gate or probe runs**, and nobody reports a result before the real exit code
  exists. Commit first, then gate.
- **One full gate per machine at a time.** Gates share ports, container names and CPU. A running gate does not hold an
  implementer or reviewer; only a second gate waits. Check the machine is on mains power (laptops throttle on battery
  and timing tests go red).
- **Slow sign-off tests** guarding rare, stable failures (startup refusals and the like) can carry a tag that leaves them
  out of normal gates; they run in the phase sweep, before a release, and whenever a batch touches what they guard.

### Reading a gate result

A gate claim counts only when all hold:

1. Its line is in the gates log: exit 0, `clean`, at **exactly** the pushed commit.
2. The counts in its output file match your prediction exactly. Predict before reading; a surplus is as suspicious
   as a shortfall.
3. No sign that tests silently did not run (each stack has its tell; list yours in the testing file).
4. The lanes are what the diff selects.
5. The machine did not sleep during the run.

A runner's "exit 0" without a line means the gate has not finished. (On the reference project, a runner reported a pass
with invented counts well before the real line appeared.) To tell hung from slow, watch the output file grow.

## 8.5 When end-to-end runs

The orchestrator decides per batch and writes the reason in the brief and the gate brief. Run the end-to-end lane in the
merge gate when the batch touched something only a live stack exercises:

- screens;
- host composition, startup, configuration or authentication (what in-process test hosts stub out);
- the development stack (containers, proxies, secrets stand-ins);
- the seed data the end-to-end tests run against;
- text or shapes a screen renders that no typecheck sees;
- anything else where the contract checks are blind; say what the risk is.

Otherwise it is not needed: an API snapshot diff, a typecheck against the regenerated client and integration tests
through the real HTTP pipeline check the contract and behavior. The phase sweep always runs end-to-end.

An edited browser test is proved by running it, and the merge gate is that run; the implementer does not run a full
end-to-end gate just for an edited spec. If the merge gate then fails on the spec, the fix is proved by one narrowed
run of the lanes it touched.

Data profiles no gate seeds (a demo profile, a large performance profile) need their own check when a batch can break
them: one seed of that profile at the merge, from a second fresh clone.

## 8.6 When something fails

**Before the first test lane** (container engine, package restore, build, migration validation, container start):
infrastructure. Retry up to twice, a minute apart, alone on the machine. Two failed retries: stop and report.

**In a test lane:** re-run **only the failing tests**, filtered, on the same tree.

| Re-run result | Ruling |
|---|---|
| Fails again | Real. A triage agent reproduces and root-causes it; a fix brief follows from its finding |
| Passes, test in the batch's own modules | Still the batch's defect (a flaky test it wrote). Fix brief |
| Passes, test outside the batch | Accept; add it to the flake register; if the failure stopped the gate, run the steps it never reached on the same clone; all green → merge |

**After a red gate, re-check proportionally.** The fix's own tests plus one run of its area; then only the lanes the fix
touched plus the lanes the red gate never reached, on the fixed commit. Lanes that passed and the fix does not touch are
not repeated. A change to core code, migrations, seeding or gate machinery still takes the full gate.

**Before blaming a machine,** name which process ran out of what, where it runs and which ceiling it hit, and read the
actual error text. A theory that cannot explain why the other machine works is not a diagnosis. Assert a setting is
live, not merely present in a file. Change one variable at a time.

## 8.7 The flake register

Every test that failed and then passed alone, outside the batch: test, date, gate, full error text, status. Capture the
error text; it is what makes the flake fixable. Every entry is scheduled for a fix; one that fires twice is not
tolerated. Recurring causes on the reference project, worth checking for in any stack: tests that assert on shared tables other tests write;
dates derived from the wall clock or shifted from a shared counter; order-dependent class execution that differs by
build path; timing-sensitive front-end tests under load; plan or performance assertions over tables other tests leave
rows in.

## 8.8 Determinism and seeding

- Ban the statically catchable causes of non-determinism in the data generator with an architecture test (new random
  ids, wall-clock reads, unseeded random), run on every batch.
- The full "seed twice and compare" runs at the phase sweep, with a comparison tool that knows which columns record
  *when* something happened. Per batch only when seeding is the batch's main deliverable (implementer seeds twice;
  reviewer seeds once from its fresh clone and compares with the implementer's dump).
- Reseed before any test that reads the seeded database; a stale database fails in ways that look like regressions.

## 8.9 Test-writing rules that held up

- Assert exact values; cover denied and negative cases; a violating write for every new database invariant;
  concurrency where it matters. Flag tautologies, over-mocking and "greater than zero" assertions.
- A test owns only its own data. A test of an organization-wide job asserts only on its own scope's rows, never on run
  totals.
- Keep the output of every filtered integration run in a file the log names; a dropped output once made a flake
  impossible to identify.
- Performance plan tests seed enough volume (including rows the statement does not ask for) that the planner's choice
  does not depend on what earlier tests left; assert the defect (a table read once per item), not one join shape among
  good plans ([09](09-performance-and-scale.md)).
