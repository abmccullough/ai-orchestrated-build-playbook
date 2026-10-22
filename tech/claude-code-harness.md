# Claude Code harness

**Applies when your stack uses Claude Code as the orchestrator and dispatches subagents.** For this playbook, that
means always.

## Agent definitions

- **One file per role** in `.claude/agents/<name>.md`, with frontmatter for `name`, `description`, `model`, `effort`
  and `tools`. The reference project had `implementer-opus`, `reviewer-opus`, `design-brief-writer` (all Opus at medium effort),
  `runner` (Haiku), and Sonnet variants that were later retired.
- **Pin the model id** (for example `claude-opus-5-5`), not an alias, so an experiment compares like with like.
- **The Agent tool takes no effort setting.** To run the same role at a different model or effort, copy the
  definition under a new name with identical instructions.
- **Measure before standardizing.** The reference project logged tokens, tool calls, wall time and outcome for each dispatch over a
  short trial. Opus reviews were markedly faster and found test blind spots with probes. A cost
  analysis then moved every implementer and reviewer to Opus. Runners stayed on Haiku.
- **Limit a cheap model to fully specified, mechanical work** (gates, pin bumps, regeneration), and never trust its
  summary of a gate. See the addendum, §1.
- **Start the session from the workspace root.** A session started from a parent folder loads neither the project's
  `.claude/agents/` nor its `CLAUDE.md`. Dispatches then fail with "agent type not found," and the only fix is a
  restart from the right folder.
- **Tool lists:** reviewers got the same read and search tools as implementers. They are told by their instructions,
  not their tools, never to edit during a review pass.

## Prompt cache TTL

- `subagentPromptCacheTtl: "1h"` in user `settings.json` (or the environment variable
  `CLAUDE_CODE_SUBAGENT_PROMPT_CACHE_TTL`) covers subagents, background and helper requests. The main conversation
  uses `promptCacheTtl`.
- **Why:** gates took many minutes, well past the 5-minute default, so every gate return re-wrote the agent's whole
  context as a fresh cache write. The owner spotted this from heavy cache-write usage.
- The 1-hour setting is ignored while the subscription is in overage. A jump in cache usage then is the setting being
  bypassed, not a regression.
- The TTL also sets the "warm agent" window. Resume an earlier agent for a fix only within about an hour of its last
  turn. Otherwise dispatch fresh.

## Long commands, background runs and timeouts

- **The Bash and PowerShell tools move any command past 600 s into the background and return.** A gate or seed can't
  be held inside one call. That's the harness, not agent disobedience.
- So "foreground" means: don't touch the checkout while the command runs, and don't end the turn until its real
  result exists. Never report on state a long command was still producing. One runner dumped a database two minutes
  into an eight-minute seed and reported a determinism failure.
- **Runner gate briefs say exactly:** call the gate script with `run_in_background: true` and `timeout: 7200000`.
  Never write "timeout 600000" anywhere in a gate brief. A Haiku runner took the 10-minute figure, and the gate was
  killed mid-run and restarted.
- Orchestrator dispatches run in the background, which is where the parallelism is real.
- A tool for an outside model that ran longer than a background shell allowed was launched in a detached window by
  script instead.

## Permissions

- **Sandbox or classifier denials inside a subagent never reach the user.** When an agent reports a denied action, or
  a brief will need one, run that command in the main session so an approval prompt (or the classifier) can decide.
  Most decisions are made by the classifier, not a human click. If the top level is denied too, record it and leave
  it for the user. Never retry through another tool.
- **Project `settings.json`** in the reference project carried:
  - `env`: `MSBUILDDISABLENODEREUSE=1` for every agent shell (see `dotnet.md`).
  - `allow`: `git push origin *` (Bash and PowerShell). The user added `git merge` and `git push` allow rules once
    the orchestrator began merging gated batches itself.
  - `deny`: every force-push form (`--force`, `-f`, with and without `origin`), for both shells.

## Messaging and multiple sessions

- **A queued message to a subagent survives its handback.** The agent resumes and does the added work. Treat the
  addition as owned by that agent, and check its checkout before dispatching anyone else there.
- **Resume the work, not the agent.** After a pause, dispatch a fresh agent with "resume from the progress log."
  Don't message the old agent or reopen an old session. That reloads and re-caches its whole context.
- **Every dispatch names a progress log** that the agent creates first and keeps current, with its final assumptions
  and questions verbatim. Read the log, not the transcript.
- **Cross-machine relay.** A second workstation ran its own top-level session (a Remote Control session), reached from
  the orchestrator by `SendMessage`. Lessons:
  - The send result says "not confirmed read." Open every request by asking for an acknowledgement.
  - The relay sends each task to a fresh subagent and never works in its own long-lived context.
  - A new session on either side gets a new address. Tell the peer the new address, or fall back to watching for
    agreed markers in a committed progress file after `git pull`.
  - The relay commits and pushes its progress logs by path. Its local, git-ignored gate log never travels, so it
    quotes its gate line verbatim.
  - The orchestrator plans, reviews and merges. The relay only implements or gates.

## Usage limits and idling

- Above 80 % of the five-hour usage window, run one agent at a time. An agent that hits the limit stalls for up to
  five hours mid-batch. The orchestrator can't read usage itself. It relies on `/usage` output the user runs, or on an
  agent failing at the limit.
- The reference project ran at most a few concurrent agents on the main workstation. When slots were contended, the longer critical
  path (the pipeline lane) went first.
- When only waiting, end the turn with a one-line status; don't fill the wait with speculative reading. To shed a
  heavy context, hand off to a fresh session (`/clear` + "read the top of the session log"; see
  `../11-context-management.md`); `/compact` only for a short wait inside one task. The orchestrator can't do either
  itself.
- For unattended nights, the user added an hourly `/loop` check as a backstop, because a dead subagent sends no
  notification at all.

## Code index (Repowise)

- A Repowise MCP index helped agents find code. `repowise init` in a new worktree pulls from the main index in about
  a minute (the reference project's worktree script ran it).
- `init` writes `.claude/CLAUDE.md`, `.mcp.json`, `.repowise/` and `.vscode/` into the checkout. List these in the
  repository's `.git/info/exclude`, which worktrees share. Otherwise the gate script sees untracked files and records
  the tree as DIRTY.
- Fresh clones used for review or gating don't have that exclude, so don't run `init` in them before a gate.
