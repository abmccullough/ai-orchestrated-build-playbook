# Testing discipline

**This file is the current testing rule.** Decision entries cited beside each rule are its history and reasons. When a
testing decision is taken, or a gate, triage or review teaches something, **edit this file in the same commit**. If this
file and a decision entry disagree, this file is right; fix whichever is wrong.

Read it before writing any brief, dispatching any gate or test run, or ruling on any test result.

---

## 1. Principles

0. **Every test run serves a purpose, proportional to the risk of the change.** Before any run, name what it could
   catch that nothing recorded covers. If nothing, skip it and record why. This rule outranks the rest.
1. **A test must be able to fail**: seen failing for the right reason against broken code, then passing. A test that
   cannot fail is deleted.
2. **A report is not evidence.** A gate counts only as its `progress/gates.log` line. Values in reports are read from
   the commit. Runners report observations, never causes.
3. **A full run belongs where two lines of work combine**: the merge.
4. **A gate is never re-run to get past a failure.** Failing tests re-run alone; the result is triaged.
5. **A partial gate is not a gate.**
6. <environment pins: the one database version, runtime versions, etc.>

## 2. Who runs what

| Role | Runs | Does not run |
|---|---|---|
| Implementer | Build; whole fast suites (<names, ~duration>); integration filtered to touched modules and consumers; front-end lint/typecheck/unit if changed; fail checks | Full gate or end-to-end, unless it judges one necessary and says why |
| Reviewer | Fresh clone: build, whole fast suites; tests the implementer did not run; fail checks only where missing, off target or weak | Any gate; a repeat of the implementer's set |
| Fixer | Each finding's tests with fail checks; touched module's tests; fast suites | Any gate |
| Re-reviewer | Fixed findings only (and only when required: security, authorization, data integrity, migrations, seeding, broad) | Any gate |
| Merge runner | One full gate per batch on the merge commit, fresh clone, end-to-end when §3 says so | Diagnosis |
| Phase sweep runner | Every repository's full gate from fresh `main`, with end-to-end; determinism check; sign-off tests | Diagnosis |
| Triage agent | Reproduce, root-cause, report | Edits, fixes |

**Gate skips** (each recorded in the ledger with the reason and the next gate that covers it): nothing to combine
(commit already gated, `main` its ancestor); documentation only; pin bump to identical code; test-only fix to wrong
tests. **Combined gates:** <rules>.

## 3. When end-to-end runs

The orchestrator decides per batch and writes the reason. Run it when the batch touched: screens; startup,
configuration or authentication; the development stack; the seed data end-to-end runs against; <project-specific>.
Otherwise not. The phase sweep always runs it. Constraints: <one live stack per machine; …>.

## 4. Seeding and determinism

<what the static determinism scan bans; when seed-twice-and-compare runs; profiles and their checks>

## 5. When something fails

- **Before the first test lane:** infrastructure; retry up to twice, a minute apart, alone on the machine.
- **In a test lane:** re-run only the failing tests on the same tree.

| Re-run result | Ruling |
|---|---|
| Fails again | Real: triage agent, then fix brief |
| Passes, in the batch's modules | The batch's flaky test: fix brief |
| Passes, outside the batch | Flake register; run the steps the gate never reached; all green → merge |

- **After a red gate:** the fix's own tests plus its area; re-check only lanes the fix touched plus lanes never reached.

## 6. Reading a gate result

A gate claim counts only when: its line is in `progress/gates.log` (exit 0, `clean`, exactly the pushed SHA); the
output's counts match the prediction exactly; <stack-specific "tests silently didn't run" checks are 0>; the lanes are
what the diff selects; the machine did not sleep during the run.

## 7. Mechanics

<gate wrapper command and its flags; filtered-run commands; fresh-clone prerequisites; container rules; probe-restore
rules; lessons from gates and triages, each dated with the batch that found it>
