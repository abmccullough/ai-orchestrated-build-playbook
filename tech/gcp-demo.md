# Shared demo on one cloud VM

**Applies when your stack needs a shared demo that invited people can use, hosted on a single cloud VM.** The reference project used a
Google Cloud free trial ($300 in credits, 90 days), Docker Compose and the existing dev stack, with synthetic data
only. The demo ran as a Dev environment, so persona sign-in worked, and connected to no production system.

## Shape

- **One VM** (e2-standard-2: 2 vCPU, 8 GB, 50 GB balanced disk, Ubuntu LTS, static IP). The estimate was about
  $59 a month. Real use was $25 in the first two weeks: $19 compute, $5 AI calls.
- **A front proxy** (Caddy, its own Compose project) owns ports 80 and 443. It gets real Let's Encrypt certificates
  for `demo.<ip-with-dashes>.sslip.io` and `perf.<...>` (no domain needed; a real domain later is a one-line change),
  puts **basic auth** (bcrypt hash on the VM only) in front of everything, and proxies to each stack's plain-HTTP edge
  port.
- **Two stacks:** the demo (always up), and a performance-scale stack restored from a database dump, started on
  request. The 5 GB performance data set was never seeded on the VM. Seeding it takes 146 minutes and 41 GB, so a
  0.24 GB dump is copied and restored instead.
- The VM builds from source. Production images weren't needed for a Dev demo.

## What agents build, and what the owner runs

- Agents write the scripts and the README (`deploy/gcp-demo/`). The owner creates the project and billing, runs the
  `gcloud` lines, and SSHes in to run setup. **Agents never hold cloud credentials.**
- Account-specific values (billing account, IP) live in a git-ignored `.env` at the workspace root.
- Repositories are cloned on the VM with a **fine-grained GitHub token**: the two repositories only, Contents
  read-only, 90-day expiry.
- `vm-setup.sh` is idempotent and refuses to reseed an existing database. Its first run stops after adding the user to
  the docker group, which is expected: log out, back in, and run it again inside `tmux`, so a dropped SSH session
  doesn't kill it.
- `vm-update.sh` pulls `main` and the pinned dependency, rebuilds, and brings each stack up so the migrator applies
  new changesets to the existing data. Never a reseed.
- `demo-reset.sh` reseeds after a typed confirmation and discards users' changes.
- Later, one PowerShell wrapper on the PC (`demo.ps1 update | reset | status | logs`) replaced SSH for routine work.

## Rules that kept the data

- **Never run `down`, `reset` or the seed by hand on the VM.** They delete the data. The scripts use `stop` and
  `start` only.
- Daily disk snapshots, kept 7 days (`gcloud compute resource-policies create snapshot-schedule ...`).
- A budget alert at 25 %, 50 % and 100 % of $200. Optionally, an instance schedule that stops the VM at night.
- Demo dates drift from the seed day. The owner accepted the drift, and reset brings the dates back (and discards
  changes). New seeded scenarios appear only after a reset. Updates bring code and schema, not new demo records.

## Things that bit

- **The first update after a script change runs the old script.** `vm-update.sh` pulls itself, so a first-run
  failure in a fixed step was cured by a second run.
- **The Vault agent stalls after VM sleep or stop.** It shows as `28P01` in the API log. Restart the agent container.
- **A deploy that changes a response's shape needs the build id in the ETag.** Reference data revalidated by ETag kept
  serving the old shape.
- **A profile no gate seeds can break silently.** The demo's first setup failed on `main` while every gate passed,
  because gates seed the standard profile. Changes that can break the demo profile now run one demo seed.
- **The demo is behind auth (401).** Agents read it only through the owner's signed-in browser session, and read the
  API logs through the wrapper script. The owner signs in. An in-app persona picker behind the password may be used.
- **A browser tab can freeze** and its requests never reach the server. Check the API log before blaming the app.
- **Running a reset from the Claude Code `!` prompt gets no stdin.** Pipe the confirmation in (`echo reset | ...`).
  See `windows-powershell.md`. A reset took about 25 minutes with the larger demo.
- **`gcloud` on PowerShell:** quote comma lists, and copy local files from their own folder (a `C:` path is read as a
  host).

## AI model costs and limits

- The demo's assistant called a hosted model (Vertex AI Gemini) billed to the same trial. Vertex had no per-project
  daily quota for those models, only shared-pool 429s. The demo's own limits (per-user day budgets, 5 a minute) or
  the trial's credits and expiry would lock users out first.
- Measurement runs against the real model were the heaviest spend, and still well inside the trial.
- If the demo must outlive the trial, upgrade the billing account before the trial ends.
