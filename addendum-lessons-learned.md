# Addendum: lessons learned

What went wrong while building the reference project, and the rule each mistake left behind. The core process files
say what to do. This file says what not to do, and why.

Each entry has three parts:
- **Happened:** the incident, kept short.
- **Rule:** what to do from now on.
- **Why:** the reason the rule holds beyond this one case.

Technology-specific traps (Windows paths, Docker, PostgreSQL planner behavior, and so on) are in `tech/`.

---

## 1. Orchestration and agents

**A queued message woke a finished agent, and two agents shared a checkout.**
- Happened: the orchestrator messaged a running implementer with an extra item. The agent handed back its report
  without the item, so the orchestrator sent a second implementer into the same worktree. The first agent had
  already picked up the message and was editing the same files.
- Rule: a message to an agent belongs to that agent, even after it reports "done." Before you dispatch anyone else
  into its checkout, check `git status` there and the tail of its progress log, or ask the agent.
- Why: a handback doesn't mean the agent has stopped. Two writers in one checkout corrupt each other's work without
  any error.

**Resuming an old agent re-cached its whole context.**
- Happened: after a usage pause, continuing work by messaging the paused agent would have reloaded and re-cached its
  long context.
- Rule: resume the work, not the agent. Start a fresh, cold agent on the brief and the progress log. Resume the old
  agent only when it finished less than an hour ago (its cache is still warm), its run was ordinary, and the fix is
  quick. If you're unsure, start fresh.
- Why: resuming saves money only while the prompt cache is warm. After that, the old context costs more than a
  cold start and carries the old run's clutter.

**A dead agent sends no handback.**
- Happened: overnight runs halted more than once. Agents died or stalled (a runner sat idle for about 45 minutes) and
  no notification ever arrived. The orchestrator would have waited all night.
- Rule: set an hourly check, and on each tick look for real activity: gate log files growing, test containers up,
  the same container id across ticks. If two ticks in a row show nothing changed, treat the agent as halted and
  investigate, whatever its progress log says.
- Why: a progress log that says "gate running" shows only what the agent believed when it last wrote. Stale state
  looks exactly like live state.

**An agent given relative paths wrote into the primary checkout.**
- Happened: a dispatch used relative paths, and the agent worked in the main clone instead of its worktree.
- Rule: dispatches and briefs give absolute paths only, including the checkout path.
- Why: an agent's working directory isn't the one you think it is.

**A cheap model's report was invented.**
- Happened: a Haiku runner reported "exit 0" with a duration and made-up counts while its gate was still in the
  integration lane. The real result arrived more than half an hour later.
- Rule: a gate claim counts only as the line the gate script writes to its log. If there's no line, the gate isn't
  finished, whatever the report says. Runners report what they ran and saw, never a cause.
- Why: the gate script records the commit, tree state and exit code itself, so it can't be talked into a pass.

**A runner gave a cause, and the cause was wrong.**
- Happened: two checkpoint runners did the mechanical work correctly and then drew wrong conclusions from it.
- Rule: runner reports have no "root cause," "regression" or "action required" sections. They say "test X failed
  with this message." Only the orchestrator, or a triage agent, states a cause.
- Why: a confident wrong diagnosis in a report steers every decision after it.

**An outside model's fail-check evidence was fabricated.**
- Happened (more than once): an implementer run on a different model pasted fail-check output that wasn't literal
  (half the checks on one batch). The tests themselves were genuine.
- Rule: a reviewer checks each pasted fail check against the commit and re-runs any that don't match. A fix that
  depends on diagnosis goes to an in-session model, or the brief names the fix's shape, not just the symptom.
- Why: one outside fix "cleared the error" by deleting rows by hand-copied ids instead of fixing the shared-name
  collision that caused it.

**Idle groundwork inflated the orchestrator's context.**
- Happened: while waiting on agents, the orchestrator pre-read design sections for a batch whose inputs depended on
  an unfinished agent. Its context grew and forced a compaction mid-build.
- Rule: when you're only waiting, stop. Give a one-line status and suggest compacting. Don't prepare work that
  depends on a report that hasn't arrived.
- Why: the orchestrator's context is the most expensive context in the system, and it carries over into every later
  step.

**Big implementer dispatches bloated their own context.**
- Rule: split a big batch into small implementer dispatches, run one after another on one branch. Use one reviewer
  per area to cover them together, and one merge gate for the group.
- Why: a reviewer of one area loads the same context either way, so paying for it once is cheaper. A small
  implementer stays sharp.

**A batch id was reused, and a merged batch's brief was overwritten.**
- Happened: a new batch was given the id of an already-merged batch, which overwrote that batch's brief. The same
  day, a draft reused a range of existing ids. It was caught only because `git status` showed the brief as
  "modified," not "new."
- Rule: before naming a batch, search the ledger and the briefs folder for the id. If a write reports "updated" for
  a file you meant to create, stop and look.
- Why: ids are the keys to every log, brief and ledger row. A collision silently merges two histories.

**The workstation's other session started in the wrong folder and found no agents.**
- Happened: a session started from the parent folder didn't load the project's agent definitions, and the first
  dispatch failed with "agent type not found."
- Rule: start every orchestrator or relay session from the workspace root. See `tech/claude-code-harness.md`.

**Silence from another workstation was read as progress.**
- Happened: cross-session messages report nothing back, and a message that never arrived looks exactly like a long
  gate that is still running.
- Rule: every cross-session request begins by asking for an acknowledgement ("ack: <task>, starting"). Silence isn't
  agreement or progress.

**The relay session ran work in its own context.**
- Rule: a top-level relay session on a second machine sends every task to a fresh subagent. It never implements,
  reviews or gates in its own context, and it relays agents' questions instead of answering them.
- Why: the relay's context lasts across tasks, but an agent's context ends with its task.

---

## 2. Merging and git

**A "refusal test" pushed to main.**
- Happened: while testing the push-merged-main script, a case expected to be refused (the fast-forward check) passed
  and pushed a batch to `main` unasked. A partial gate also counted as green in that run.
- Rule: any script whose last step reaches outward (push, deploy, delete) gets a `-DryRun` switch before its first
  test, and every test uses it. Reason through each guard for the test input before running anything that can reach
  a remote.
- Why: you can't "test the refusal path" safely on a real remote. If the guard has a bug, the test is the incident.

**A handoff's "nothing to merge" let ungated work onto a branch.**
- Happened: a session handoff said a batch branch already contained the target branch's latest commit. It didn't, and
  the ungated combined tree was pushed. A post-merge end-to-end gate covered it afterward (green).
- Rule: before merging, run `git merge-base --is-ancestor <target> <gated SHA>` and the tree check
  (`git diff <gated SHA> HEAD` is empty) before you push, never chained after an echo. A brief says "contains main
  (checked)" only after the check has actually run.
- Why: a handoff note is a claim. Ancestry is a fact you can check in one command.

**A stale ref made the "tree equals gated SHA" check pass for the wrong reason.**
- Rule: `git fetch origin` first, every time. Then merge `origin/main` into the batch branch with a merge commit
  (never a rebase), gate that commit, merge with `--no-ff`, and confirm the diff against the gated SHA is empty.
- Why: without the fetch, an empty diff can just mean you compared against an old copy.

**A blanket commit swept an agent's in-progress edit into an orchestrator commit.**
- Happened: `git commit -qam` in the workspace picked up a running agent's half-finished edit to the gate script.
  Later, `git add progress` swept over a hundred old untracked logs into one commit.
- Rule: in a shared workspace, commit named paths only. Never `commit -a` or `add -A`.
- Why: agents keep uncommitted work in the same tree you commit from.

**A branch missing a merged changeset couldn't gate at all.**
- Happened: a branch whose base predated another merged batch failed the changelog-immutability step in under a
  minute.
- Rule: bring `main` into the branch before review, not after. When a batch is about to land, merge it into the next
  batch early, off the gate's path, so the next gate runs on a resolved tree.
- Why: the failure says nothing about the batch, and it costs a gate slot to find out.

**A ledger edit matched too much.**
- Happened: an edit that matched `| B12 ` as a prefix also changed follow-up rows. Python string edits turned `\t` in
  Windows paths into tabs in two ledger rows in one evening.
- Rule: match a ledger row by its full key cell (`| B12 | `), and make edits that include Windows paths with an
  exact-string edit tool, not a script.

**Merges were blocked by the permission classifier.**
- Happened: for weeks, every merge and push to `main` was blocked, so the owner ran each one by hand. Allow rules for
  `git merge` and `git push` fixed it. Force-push patterns stay denied.
- Rule: if a merge or push is blocked, stop and hand the command to the owner. Never work around a block through
  another tool. Unattended, keep the verified local merge and list the unpushed merges for the morning.

**Two machines appended to the same gate log and the pull conflicted.**
- Rule: when two machines append to one log, resolve the conflict by keeping both sets of lines, sorted by timestamp.
  A second machine's gate log never leaves that machine. Its relay quotes the line verbatim, and the orchestrator
  checks the SHA against the pushed branch.

---

## 3. Testing and gates

**A gate was re-run to get past a failure.**
- Rule: never. Re-run only the failing tests, filtered, on the same tree. If it fails again, it's real: triage it and
  write a fix brief. If it passes and the test is in the batch's own code, it's still the batch's flaky test: fix it.
  If it passes and the test is outside the batch, log it in the flake register and proceed. Infrastructure failures
  before the first test lane (Docker down, a lock left behind) may retry up to twice.
- Why: on one busy day, well over a third of the gates failed, mostly on faults unrelated to the batch. Re-running
  hides the real defects among them and doesn't fix any of the unrelated ones.

**The flake register had no teeth at first.**
- Rule: every flake entry gets a scheduled fix. If it fires a second time, it's a fix row, not a tolerated
  annoyance. Capture the failing output the first time: one assistant flake couldn't be identified because the
  run's output was dropped.

**Gates ran far too often.**
- Happened: one workstation ran dozens of gates (hours of gate time) in a day, and the same commit was routinely
  gated three or four times.
- Rule: one full gate per batch, at the merge, from a fresh clone. Implementers and reviewers run focused, filtered
  tests. Batches that are ready together share one gate commit and one run.
- Why: a full run is only needed where two lines of work combine.

**A half-hour end-to-end gate could catch nothing.**
- Happened: a full gate ran on a dependency pin bump to code identical to an already-gated commit. The owner asked
  what it was for.
- Rule, the owner's first testing rule: before any run, name what it could catch that nothing already recorded
  covers. If the answer is nothing, skip it and record why. Scale the run to the change.

**Reviewers re-ran the implementer's tests.**
- Happened: the owner called it "a big time waster."
- Rule: a reviewer repeats a run only when the repeat can catch something new, such as a fresh-clone build or tests
  the implementer didn't run. It reads the implementer's fail checks (the break, the failing output, the restore)
  instead of repeating them.

**A partial gate was treated as a gate.**
- Happened: a gate exited 1 with every test passing, because the test reporter ran out of memory while summarizing
  about 1,000 results.
- Rule: a gate that stops early proves nothing about the steps it never reached. After a red gate, re-run the lanes
  it never reached plus the lanes the fix touched, and record both log lines.

**A runner's gate was killed by a 10-minute timeout.**
- Happened: the brief said both "timeout 600000 per call" and "run in background." The runner used the 10-minute
  limit, and the gate was killed mid-run and restarted from scratch.
- Rule: gate briefs say exactly "run in background, timeout 7200000," and never mention 600000. See
  `tech/claude-code-harness.md`.

**Two gates on one machine ran out of memory.**
- Happened: an end-to-end gate beside a reviewer's test runs took nearly half an hour, and its test host threw an
  out-of-memory error after every test had passed.
- Rule: one full gate per machine at a time, enforced with a machine-wide lock. A running gate is not a reason to hold
  implementers or reviewers (the owner's ruling); only a second gate waits.
- Why: the stricter "nothing beside a gate" reading left agent slots idle for no benefit.

**A gate launched from the wrong shell faked a pass.**
- Happened: Git Bash stripped backslashes from Windows paths, the gate never started, and the notification reported
  the wrapper's exit 0.
- Rule: never accept a notification's exit code as the gate's. Look for the gate script's own log line. See
  `tech/windows-powershell.md`.

**A runner wrote the gate log itself.**
- Happened: one runner redirected output into the gate log, which destroyed its history. Another hand-wrote a line in
  its own format.
- Rule: briefs say "never write to the gate log; quote the script's line verbatim," and the orchestrator checks the
  quoted line's shape.

**A runner reported a red gate from the previous run's log.**
- Rule: check every gate report against the new log line's timestamp and SHA before acting. A log written step by
  step looks hung during a long step, so check the processes before calling a run hung.

**A runner kept going after a failed step.**
- Happened: after a failed seed, a runner still started the stack and ran the demo preparation.
- Rule: runner briefs say **stop at the first failure** in bold, and the orchestrator reads the log, not its summary.

**A stale rule in the instructions caused a retired test run.**
- Happened: the orchestrator instructions still required a per-batch "seed twice and diff" that a decision had
  retired the day before, and one was dispatched.
- Rule: keep one current testing-rules file. Decision records are history and point to it. Every testing decision or
  lesson edits that file in the same commit. "Random decision records get lost" (the owner).

**A skipped end-to-end lane left `main` red.**
- Happened: a batch changed which data a report screen renders and was gated without end-to-end tests. A screen spec
  broke on `main`.
- Rule: the orchestrator decides end-to-end per batch and writes down the reason. Anything a screen renders whose
  text no typecheck sees takes end-to-end.

**A profile no gate exercised broke the demo.**
- Happened: the demo's first cloud setup failed on `main`. Every gate passed because gates seed only the standard
  data profile.
- Rule: if a change can break a data profile that no gate seeds, it runs that profile's seed once to completion, on
  the gated commit, from a second fresh clone.

**Tests passed alone and failed in the full run.**
- Happened (many times): shared date counters, a global watermark, organization-wide jobs, class order that differs
  between build paths, and plan tests sensitive to other tests' leftover rows.
- Rule: no test owns a shared table or a shared date. Assert only on your own rows. To reproduce cheaply, run the
  test's folder from a second path.

**A test couldn't fail.**
- Rule: a test for an acceptance item or a fixed finding counts only after it has been seen failing for the right
  reason against broken code, then passing. A test that can't fail is deleted, not kept "for defense in depth."

**A red gate was blamed on the batch while the laptop was on battery.**
- Happened: timing-bound tests failed after about 20 minutes on battery. Earlier gates were spoiled by sleep.
- Rule: check AC power and recorded suspend time before a gate and before triaging one. A run with more than 60 s of
  suspend is not evidence.

---

## 4. Performance

**Small seeds hid costs that grow with data.**
- Happened: N+1 reads on list pages, a whole-table rebuild triggered by a one-row change (most of a 30 s timeout at
  mid scale), aggregation in the API instead of SQL, a hot lookup with no index, and a row per date instead of a
  range (gigabytes reduced to tens of megabytes).
- Rule: build for 100x to 1000x local data. A cost that grows with data and has a known fix (an index, batching, set
  SQL, ranges) is fixed when it's seen, even well inside the targets. "Close" and "watch" are only for costs with no
  growth term.
- Why: local margins say little about production scale. By the time it shows, it's a habit repeated across screens.

**A list column that needed per-row lookups.**
- Rule: a list column that can only be filled by reaching past the list's own row needs a reason to exist. Raise it as
  a design defect instead of routing around it silently.

**Plan tests asserted the wrong thing.**
- Happened: plan tests passed until other tests' rows tipped the planner, failed by neighbor, or asserted one join
  choice among good plans.
- Rule: assert the defect itself (a table read once per item, a whole-set read), not one plan shape. Seed your own
  volume, including the rows the statement does not ask for. Measure rows into the expensive step, not only time.
  Details are in `tech/postgresql.md`.

**A fix made things slower.**
- Happened: keying a history read by a set of record ids was twice as slow when the keys were a large share of the
  table.
- Rule: measure before you adopt a pattern as a rule, and record the counter-case beside the rule.

Keep a living performance log: one section per pattern, with what it looks like, why it costs what it costs, the
rule that catches it, and every instance found. Brief writers and reviewers consult it whenever work touches a
list, a job or a query.

---

## 5. Briefs

**A brief template missing one step cost a 25-minute gate.**
- Happened: an end-to-end gate failed at the reseed because a pipeline batch had merged and the brief, written from an
  old template, didn't include the pin update. The rule existed; the template lacked it.
- Rule: before every gate brief, compare the dependency pin with the dependency's `main`. Put standing checks in the
  template itself, not only in the rules.

**Briefs generated by a script came out corrupted.**
- Happened (twice): briefs built with inline Python string replacements turned `\t`, `\a` and `\r` in Windows paths
  into TAB, BEL and CR.
- Rule: write each brief in full with a file-writing tool, even when it is mostly a copy.

**Runners need everything spelled out.**
- Happened: a runner ran a repository script from the workspace folder and got an error that looked like a broken
  SDK. Another ran its pre-checks in the wrong directory and stopped on a false mismatch.
- Rule: runner briefs start every step with `Set-Location <clone>`, write checks as `git -C <clone> ...`, give
  explicit clone paths, and say that the runner's default directory is the workspace.

**A review brief that asserts a defect steers the reviewer.**
- Rule: a brief never states a defect as established unless the orchestrator has seen it in the tree. Read any value
  or literal a report describes from the commit (`git grep <literal> <sha>`) before you rely on it.
- Why: a report is the agent's recollection of what it built, and it can disagree with what it pushed.

**Fix briefs that depended on one agent's memory.**
- Rule: every fix brief is self-contained (path, branch, SHA, review log, the implement brief for context, and a
  fail check per item), so either a warm agent or a fresh one can take it.

**Content briefs drifted into British spelling.**
- Rule: every content brief says "American spelling."

---

## 6. Working with the owner

**Batch ids meant nothing to the owner.**
- Happened: "What is B23b, B19b. Need you to define them, I don't have them memorized."
- Rule: name a batch in plain words beside its id, every time.

**Chat recaps buried the owner.**
- Happened: multi-section status replies, during the day and overnight.
- Rule: keep chat to a line or two: a question the owner must answer, a result they asked for, or one line of status.
  Detail goes in the session file, the ledger and the progress logs, committed, for review later.

**A settled question was re-opened.**
- Happened: the orchestrator proposed code-only exception logging again after the owner had already ruled that
  diagnosability outranks data minimization in logs.
- Rule: search the decisions before proposing. If logs can't diagnose a failure, that's a defect against the existing
  ruling, not a new trade-off.

**Protective defaults added complexity the owner didn't want.**
- Happened: findings that a reporting screen revealed only aggregate counts were raised as security issues. The owner
  said the counts were already widely available.
- Rule: findings about writes, resource exhaustion, audit and integrity are fixed. Visibility edge cases that add code
  or remove a feature go to the owner with a recommendation. Don't default to "more restrictive."

**Edge cases ate fix loops.**
- Happened: a log guard took three review rounds on rare edge cases.
- Rule: build the simplest thing that works, and put edge cases in the follow-ups. Before a handoff to another team,
  everything known and fixable is fixed, not left as a follow-up.

**The two-fix-loop stop kept bringing the owner obvious fixes.**
- Happened: twice, the stop brought a clear fix that the owner approved immediately.
- Rule: there is no loop count. A clear, material fix is always made. Bring a batch to the owner only when there's no
  clear solution: the cause isn't understood, or the last fix failed and the next would be a guess.

**Phase-end checkpoints waited for a go-ahead that was arbitrary.**
- Happened: "don't wait on me for phase end. It is arbitrary anyway."
- Rule: at a phase end, run the regression sweep, write the report, push, and carry on. Still do the visual browser
  pass, because it "finds things other tests can't see." Hard stops still stop.

**Questions for an outside data owner blocked work.**
- Rule: collect questions that need real enterprise data in one file, as aggregate-only questions with a "why." Build
  against the stub meanwhile, and tell the owner when one is added.

**Advice was backwards because the ceiling wasn't checked.**
- Happened: the orchestrator told the owner to raise a VM memory cap to fix test out-of-memory failures. The failing
  process ran outside that VM, so the change would have taken memory away from it. The next theory (a fixed heap
  limit) didn't explain why the other machine worked. Then a copied "working" setting turned out never to have been
  active. The real limits were orphaned build processes and a small fixed page file.
- Rule: before prescribing a resource change, name the process, where it runs and which limit it hit. Read the actual
  error text. A theory that can't explain why the other machine works isn't a diagnosis. Check that a setting is
  live, not just present in the file. Change one variable at a time.
- Why: advice about someone else's machine is where a guess costs the most and is noticed the least. It had already
  been written into a decision record before the owner asked "why raise the memory cap?"

**Times were guessed, wrongly, about ten times in one day.**
- Happened: guesses ran up to 40 minutes fast. A gate called "silent 45 minutes" had run 13. A needless status
  request went to the other workstation, and wrong times went into the session notes.
- Rule: never estimate a time or an elapsed time. Read the clock, and subtract from a start time that was itself
  read from the clock or a log. If no start was recorded, say you don't know. Mark times "(clock read)."

**Permission denials inside subagents never reached the owner.**
- Happened: "Agent permissions don't come to me."
- Rule: when an agent reports a denied action, or a brief will need one, run that command in the main session so the
  owner, or the classifier, can approve it. Tell the owner what it is. If it's denied at the top level too, record the
  block and leave it for the owner. Never work around a denial.

**The owner signs in.**
- Rule: shared passwords for the demo site stay with the owner. The agent never enters them. Read the protected
  site through the owner's browser session.

---

## 7. Autonomy and overnight runs

**The cautious option was assumed.**
- Happened: asked how far to go overnight, the owner chose the widest option and "spend freely" with over half the
  weekly quota already used. An idle machine is the waste the owner cares about.
- Rule: before an unattended stretch, ask once with the real options, then record the answer as an authorization,
  including what it doesn't cover.
- Why: an authorization given verbally before an unattended run later looks like the assistant deciding for itself.

**Hard stops hold, whatever the grant.**
- Rule: force pushes, history rewrites, repository settings and design gaps that change behavior still wait for the
  owner. While the owner is away, decide small gaps that stay inside one batch and change nothing users see, the
  security model or the data shape. Record each as "decided while away; for review." Larger gaps block only their
  batch. On the owner's return, list everything marked for review first.

**Sessions were hard to resume cold.**
- Rule: keep a session file (what's running, each agent's brief, log and checkout, what's next) and a run-plan status
  table, both updated with every ledger update. A fresh session reads them first. Prefer handing off to a fresh
  session over carrying a huge context.

**Usage limits stalled agents mid-task.**
- Rule: above 80 % of the five-hour usage window, run one agent at a time. When pausing for a limit, ask agents to stop
  at a safe point with a WIP commit and a resume point in their log.

**Reboots killed agents.**
- Happened: a reboot killed an implementer mid-run, and a Windows update rebooted the second workstation during a
  measurement burst.
- Rule: every agent keeps its progress log current enough to resume from, and recovery starts by checking the stack,
  the lock and the settings before re-running anything.

---

## 8. Measurement campaigns

**A "fixed" setting had never been applied.**
- Happened: a VM memory-reclaim setting sat under the wrong section and was silently ignored for days. A premise
  check run before a measurement found it.
- Rule: run premise checks before a measured run. A premise check is often worth more than the run.

**Measurements need a quiet machine.**
- Rule: during a measured phase, the measuring machine runs no gate, stays on AC power and stays awake. Record the
  start and end times from the clock, and exchange them with the other machine for joins.

**Budgets and limits were raised for a test.**
- Rule: when a measurement raises a limit (a token budget, a pool size), set it back afterward, read the value back
  to confirm it, and log both steps. Throwaway logins or grants made for a measurement are dropped afterward.

**A phase turned out not to be needed.**
- Happened: the owner stopped a multi-hour phase whose question had already moved to the follow-ups.
- Rule: before a long phase, confirm with the owner that its answer is still wanted.

**A failure under load was a code defect, not load.**
- Happened: a few of the scheduled runs hit "permission denied for schema." It looked like a race, but it was a
  module-boundary violation on a rare code path.
- Rule: under load, read the error text before calling a failure infrastructure.

**The full-scale data set was expensive to make.**
- Rule: a full-scale seed (on the reference project, hours of runtime and tens of gigabytes) is made once, dumped, and
  restored from the dump. It gets light checks (completes within its limits, row counts) and no functional tests.

**A quiet-day test proved less than it seemed.**
- Rule: test at the design's realistic rate of change too. A plan test on a quiet day proves the index is usable, not
  that the check is bounded.
