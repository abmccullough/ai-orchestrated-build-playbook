# Windows and PowerShell

**Applies when your stack runs on a Windows workstation, with PowerShell 7 scripts and Git Bash beside them** (as in
Claude Code on Windows, which offers both a Bash tool and a PowerShell tool).

## Launch gates from PowerShell, never Bash

- Git Bash strips backslashes from Windows paths: `C:\work\myapp-workspace\repos\myapp` becomes
  `C:workmyapp-workspacereposmyapp`. The gate script never finds the checkout and dies before the gate starts.
- The failure looks like a pass. The task notification reports the wrapper's exit 0, not `pwsh`'s. It also leaves
  folders in the working directory named after the mangled paths (for example `C:worktmp`). Sweep for them now and
  then.
- The guard is a gate log written by the gate script itself, so a mangled run writes no line. Call scripts as
  `pwsh -NoProfile -File <script>`, using `-File` and never `-Command`.

## Paths and text

- **Never build files with Python string literals that contain Windows paths.** `\t`, `\a` and `\r` become TAB, BEL
  and CR. This corrupted two briefs and two ledger rows. Write the file with a file tool, or make an exact-string
  edit.
- **`[IO.File]` calls resolve relative paths against the process directory, not `Set-Location`.** Use absolute paths.
- **Line endings:** a large LF file (such as a project ledger) edited by a tool that writes CRLF rewrites every line in the diff.
  Preserve the file's line endings.
- **Call `pwsh` by name.** winget installs an App Execution Alias, and `C:\Program Files\PowerShell\7\pwsh.exe` may
  not exist.
- **`gcloud` and similar CLIs:** quote comma lists (`--allow="tcp:80,tcp:443"`), or PowerShell turns them into an
  array. In `scp`-style commands, a path starting with `C:` is read as a host name, so copy from the file's own
  folder.

## Shell traps

- **`python` in Git Bash can hang** on the Microsoft Store stub. Use `uv run`, or call a known interpreter.
- **`git pull` that creates a merge commit opens an editor and hangs.** Use `git pull --no-rebase --no-edit`. Wrap
  `git push` in a timeout (for example `timeout 90`).
- **`cat >> file` with no input hangs** a Bash call.
- **The `!` prefix** (the user running a command in the Claude Code prompt) runs Git Bash with no stdin. A script
  that prompts with `Read-Host` sees EOF. Give the user forward-slash paths, and pipe the confirmation in:
  `! echo reset | pwsh -NoProfile -File C:/path/to/demo.ps1 reset`.
- **Runners start in the workspace, not the clone.** Brief every step as `Set-Location <clone>; ...`, and every check
  as `git -C <clone> ...`.

## Memory: find the real ceiling

- **WSL's `.wslconfig` caps the Docker VM, not Windows processes.** Node test runners and the .NET test host run on
  the Windows side. Raising the WSL cap takes memory away from them.
- **`autoMemoryReclaim` belongs under `[experimental]`.** Under `[wsl2]` it is silently ignored. The reference project carried an inert
  copy for days. Enabling it can also break Docker in WSL, so it stayed off.
- **Check the page file and commit limit first.** One machine's fixed 2.9 GB page file gave it a commit limit 13 GB
  smaller than an identical machine's. It failed like missing RAM, and nothing done to RAM would fix it.
- After a settings change, prove the setting is live (with `wsl --shutdown` or a Docker restart, never mid-gate).

## Power and sleep

- **On battery, a laptop throttles the CPU, and timing-bound tests fail even when run alone.** Before a gate and when
  triaging a red one, check `(Get-CimInstance Win32_Battery).BatteryStatus` (2 means on AC power).
- **Sleep during a gate wedges containers and stretches timings.** The reference project's gate script records suspended time as the
  larger of (interrupt time minus unbiased interrupt time) and the Modern Standby time from System event log
  Kernel-Power 506/507. A run with more than 60 s suspended isn't evidence. On Modern Standby laptops the unbiased
  clock keeps running for most of a standby session, so read the event log too.
- **A sleeping host also stops background token renewals.** See the secrets agent in `docker-and-local-stack.md`.

## Gate lock script (pattern)

The reference project's `run-gate-locked.ps1` wrapped each repository's `verify.ps1`:
- Holds a machine-wide lock, so two gates never run together. Exit 75 means "lock held, nothing ran," so call it
  again.
- Creates its temp directory itself. Before that fix, a missing scratch root (`C:\work\tmp`) raised a `DirectoryNotFoundException`.
  That error derives from `IOException`, so the lock's catch block swallowed it and reported a false "another gate is
  running."
- Writes one line per run to a gate log:
  `start | exit N | seconds | clean or DIRTY | SHA | checkout | output file | suspended Ns | GATE-LANES: ...`. The
  full output goes to a per-run file. Machine state (free memory, WSL working set, containers, busiest processes)
  goes at the top of the output, so you can read it before blaming a slow run on the batch.
- Sets `MSBUILDDISABLENODEREUSE=1`.

## Other workstation setup

- Make your scratch root (for example `C:\work\tmp`) before the first gate on a new machine.
- Firmware virtualization: an Intel laptop reported a hypervisor running but firmware virtualization off, which is
  usually a reporting quirk. On AMD, `HypervisorPresent` read False only until the restart that `wsl --install`
  requires. Restart once before you go hunting in firmware.
- Keep a per-machine table of which tools are actually verified. A single "installed" snapshot misled a batch onto a
  machine that lacked the tool.
- If a machine's git identity is unset, agents commit with per-command `-c user.name=... -c user.email=...` and don't
  change the config.
