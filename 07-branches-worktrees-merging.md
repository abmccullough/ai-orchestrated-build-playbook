# 7. Branches, worktrees and merging

The aim: every tree that reaches `main` is exactly a tree some gate saw whole.

## 7.1 Layout

```
<workspace>/                     orchestrator session starts here; design, plan, progress
  repos/<repo>/                  primary checkout of each product repository, on main between batches
  .worktrees/<repo>--<batch>/    a worktree per parallel batch, on batch/<batch>
```

Primary checkouts and worktrees sit at the same depth so relative paths to `design/` work from either. Product
repositories are git-ignored in the workspace.

## 7.2 Serial by default

- A serial batch: in the primary checkout, `git switch main`, `git pull --ff-only`, `git switch -c batch/<id>`.
- **One branch per repository per batch.** A batch touching two repositories has a branch in each (and a worktree in
  each if parallel).
- Two agents never share a checkout. Sub-batches of one area run in sequence on the same branch, each starting from the
  previous one's pushed head.
- Between batches each primary checkout is clean and on `main`. Check at every session start.

## 7.3 Worktrees for parallel batches

- Only for batches the plan marks parallel, and only when their context does not overlap much
  ([02 §2.7](02-roles-and-agents.md)).
- Create with a script (`new-worktree <repo> <batch>`) that makes `.worktrees/<repo>--<batch>` on `batch/<batch>` from
  current `main`, and gives it its own database name and ports; the brief states them and says never to touch the
  primary checkout.
- Anything a worktree needs that is not in git (local certificates, secrets, tool indexes) is set up by the script or
  by the orchestrator before dispatch.
- The agent harness's own "isolated worktree" option failed on the reference project; prefer your own script. Re-test it now and then.
- Remove the worktree and delete the branch after merge. Check leftover worktrees against the ledger before removing.

## 7.4 Pushing

- Implementers push the batch branch **after every commit**. An unpushed commit exists on one machine; on the reference
  project two implementers were stopped mid-batch holding three unpushed commits each.
- Nobody but the orchestrator pushes `main`, and only after a green gate on the exact tree.

## 7.5 The merge sequence

1. **Confirm no agent is active** in the batch's checkout (check its log and `git status`).
2. **`git fetch origin` first.** A stale `origin/batch/<id>` (never fetched since an agent pushed it, possibly from
   another machine) sits at an old commit, so the final tree check comes out empty for the wrong reason and silently
   passes.
3. **Merge `origin/main` into the batch branch** as a merge commit. **Never rebase**: a rebase rewrites commits a
   review and progress logs refer to. Regenerate generated files that conflict (API snapshots, generated clients)
   rather than hand-editing them. Push.
4. **Gate that exact commit from a fresh clone** (a runner, [08](08-testing-discipline.md)). Fresh clone because it
   proves the pushed tree alone is complete.
5. **On green:** on `main`, `git merge --no-ff origin/batch/<id>`. Then `git diff <gated SHA> main` **must be empty**
   before pushing. Push `main`.
6. **Clean up:** remove the worktree, delete the branch.

`--no-ff` keeps each batch as one visible merge on `main`, so a batch can be found, reviewed or reverted as a unit.

### Bring main in early when it moves

If `main` gains a batch while yours is in flight, merge `main` into your branch before review, as product work (a
small fix brief if conflicts need judgment), not after. On the reference project a check that every migration on `main` also exists on
the branch failed any stale branch in under a minute, before a single test ran; a stale branch has no green gate
available to anyone.

### Collisions between parallel batches

Migration numbers, changelog order, fixture ids, generated snapshots: resolved on the later-merged batch by
renumbering or regenerating, through a fix brief, never by an unrecorded hand edit.

## 7.6 When no merge gate is needed

- **Nothing to combine:** the exact commit being merged already has a green full gate recorded and `origin/main` is its
  ancestor. Verify with `git merge-base --is-ancestor origin/main <gated SHA>` and the tree check **before** pushing;
  never trust a handoff note's "already contains main". (On the reference project, such a note was wrong, and an ungated tree reached a
  shared branch.)
- **Documentation only** (outside any test's reach): the orchestrator reads the diff.
- **A pin bump to identical code** in the other repository (the trees are equal): only an id changed.
- **A test-only fix to tests that were wrong** (the product was right), checked by their lane's compile and lint: merge
  with the tree equal to the fixed SHA; the next gate covers it.

Each skip is recorded in the ledger with its reason and the next gate or sweep that covers it.

## 7.7 Combining gates

Gates are the slowest step and run one per machine, so share them:

- **Combined gate:** two or more reviewed batches ready at about the same time are merged into one gate commit
  (merge commits, from `main`), gated once with the widest lanes any needs, and merged into `main` with the tree equal
  to the gated SHA. A red combined gate is triaged; if one batch causes it, drop that batch and gate the rest rather
  than stall all.
- **Sub-batches and same-area batches** share one gate after one review.
- **Ride-along:** a small branch whose own gate would repeat another's merges into it and both merge on that one gate.
- **Main moved after a green gate:** merge it in and re-check only the lanes the overlap touches, provided the merge is
  conflict-free and no shared file is core code, migrations, seeding or gate machinery; otherwise a full gate. Name the
  next full gate that covers the combined tree.

Worth it when the batches are ready within about one gate's length of each other; a batch that would wait longer takes
its own gate.

## 7.8 Outward scripts

Any script whose last step reaches a remote (push, deploy, delete) gets a dry-run switch before its first test run, and
every test uses it. Never "test the refusal path" for real on the assumption a check will stop it. (On the reference project, such a
test pushed a batch to `main` unasked.)

## 7.9 What needs the owner

The loop is standing authorization to commit, merge and push reviewed, gated batches and to push the workspace. Force
pushes, history rewrites, deleting repositories or branches others may use, and changing repository settings need
explicit approval. If a permission classifier blocks a merge or push, stop and hand the command to the owner; never
work around it.
