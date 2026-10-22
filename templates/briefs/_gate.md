# <BATCH-ID(s)> · <plain words> — merge gate (<full | with end-to-end>) on <machine> — runner brief

<!-- Before issuing: cross-repository pins match what this commit needs; the machine is on mains power; the container
engine is up; nothing else holds the gate lock; no other full gate on this machine. Combine other ready batches if
they can share this gate. Tool-specific commands: see the tech notes. -->

**Agent:** runner, the only gate on <machine> · **Issued:** <date> · **Log:** `progress/<ID>-gate.md`. Create it first,
keep its Status current after every step, put the report in the final entry. Read the clock for every time you write.

`batch/<ID>` is at **`<sha>`**. <It already contains `main` `<main sha>` (checked with `merge-base --is-ancestor`), so
there is nothing to merge. | Merge `origin/main` into it with a merge commit (never rebase) and push; that commit is
the GATE-SHA.> First check with `git ls-remote` that `main` still prints `<main sha>` and `batch/<ID>` prints `<sha>`.
If either does not, stop and report. **GATE-SHA = `<sha>`.**

**Never touch `main`. No commits** beyond the merge commit above, if any. Never stop a resource you did not start.
**Never write to `progress/gates.log`: only the gate script writes it.** If a command is denied permission, stop and
report it.

## Steps

1. **Prepare.**
   - Confirm mains power, the container engine responds, and no other stack is running. Report each.
   - Delete `<scratch>/<id>-gate-clone` if present; clone `batch/<ID>` there. HEAD must start with GATE-SHA.
2. **Gate.** From the clone: `<gate wrapper command> -Checkout <clone> <-E2E>`. Wait for the real exit code (if the
   tool backgrounds it, wait for the completion notice; allow up to <hours>).
   - **A failure before the first test lane is infrastructure:** retry up to twice, a minute apart.
   - **From the first test lane onward, stop at the first failure.** Do not re-run, diagnose or edit.
3. **<Optional> Extra profile check**, only if step 2 is green: <e.g. seed the demo profile from a second fresh clone,
   bring the stack up, run the prep script twice (the second must create nothing), take it down>. Report each exit code
   and the last lines; stop at the first failure and quote it.
4. **Finish.** Commit and push only `progress/<ID>-gate.md` and `progress/gates.log`, by path. If the push is rejected,
   pull with a merge (keep both sides of any `gates.log` conflict) and push again; never force.

## Report

The new `gates.log` line verbatim (SHA = GATE-SHA, tree `clean`, lanes quoted exactly); each step PASS or FAIL; every
test lane's totals; for any failure the test name, its file and the whole failure text, naming it if it is in the
ledger's §Flake register. Observations only.

<One paragraph for the orchestrator's record: what the batch changed and why this gate shape (lanes, end-to-end or not,
extra profile check or not).>
