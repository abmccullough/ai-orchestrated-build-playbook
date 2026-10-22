# Docker and the local stack

**Applies when your stack uses Docker Desktop, a Compose dev stack, Testcontainers, a local secrets agent, or local
TLS.** The reference project's stack had PostgreSQL, Vault plus a Vault agent, a mock OIDC issuer, one-shot Liquibase containers, the
API, a worker, and the web app behind Caddy.

## Is the engine actually up?

- Before a gate: `docker ps` must exit 0, and `docker info --format '{{.ServerVersion}}'` must exit 0 and print a
  version.
- **`docker info | head` only prints the client block**, so it "responds" with the engine stopped. One gate died in
  2 s after a reboot this way. A 2-second run with an unknown lane list is the signature of a preflight death.
- **Don't check `wsl -l -v`.** Recent Docker Desktop shows `docker-desktop` Stopped while the engine serves. One
  runner stopped on that false precondition. If the checks disagree, `docker run --rm <small image> --version`
  settles it.
- Starting Docker Desktop is routine gate prep. Changing its settings is not.

## One stack per machine

- **Every checkout used the same Compose project name**, so a second stack adopted and then tore down the first.
  A seed in one checkout (`down -v` across every profile) wiped an end-to-end run in another. The edge proxy also
  binds one host port.
- Rule: one end-to-end run or seed per machine at a time. The end-to-end gate refuses to start while any container of
  the project is up, and at teardown refuses to remove a stack another checkout started.
- A per-checkout project name would remove the conflict. The reference project kept the singleton and scheduled around it.

## Restart a stack, not every container

- `docker start $(docker ps -aq --filter label=com.docker.compose.project=X)` also starts the exited one-shot
  migration containers. They bind-mount the worktree's changelog and apply whatever the branch holds now. Once
  this applied a later phase's `DROP TABLE` conversion to a baseline database early.
- Restart only the services that were running (note the names before stopping), or use the stack's own `up`
  command. Before any restart, ask which migrations the changelog has that the database doesn't.
- Prefer `stop` and `start` over `down -v` wherever the data matters. In the reference stack, `down` removed volumes.

## Secrets agent stalls after sleep

- Symptom: after hours idle, the dev API returned 500 with `28P01 password authentication failed` for a dynamic
  `v_...` user. The Vault agent's last token renewal was about 9 hours earlier (host sleep), so the rendered
  credential files had expired.
- Fix: `docker restart <vault-agent container>`, then wait for fresh credential file timestamps. The API recovered
  without a restart, because new connections read the new files. The same fix applied on the demo VM.
- Check the API log for `28P01` and the agent log's last renewal before restarting anything else.

## Local certificates per checkout

- The database TLS certificates were per checkout, git-ignored, and made by a script, not by worktree creation. A
  fresh worktree's PostgreSQL init then failed (`cannot stat /certs-src/server.crt`), reported as "did not become ready
  within 60 seconds."
- Run the certificate script (idempotent) when you create any worktree that will run a stack or seed. Gates on fresh
  clones did it themselves.
- .NET doesn't resolve `*.localhost`. Use a browser or `curl.exe --resolve` for the local edge.

## Cleanup

- **Testcontainers leaves anonymous volumes.** The first sweep found 223 volumes and 10.7 GB, and a later one 329.
- `docker volume prune -f` removes anonymous unused volumes only (`-a` would include named ones). Snapshot the named
  volumes before and after: it's one command and turns "the flag looks right" into evidence. `docker builder prune -f`
  and `docker image prune -f` (dangling only) reclaimed about 1 GB more.
- **Never `docker image prune -a`.** It drops digest-pinned images and forces re-pulls.
- Prune in a quiet window, not during a gate. A build or container start can race a prune.
- **Only the orchestrator prunes or stops project stacks**, and only on a machine with no other Docker use, with the
  owner's say-so. Agents running in parallel never run unscoped `docker rm`, `stop`, `kill` or `prune`, never stop a
  container their own run didn't start, and never `docker run`, not even for a probe. A probe that needs a database is
  a temporary test on the Testcontainers fixtures.
- **A stale Compose network** (`<project>_default`) with no containers caused `UnknownHostException: postgres` in a
  migration. `docker network rm` it and re-run. That's infrastructure, not code.

## Testcontainers as a heartbeat

- A live gate shows test containers plus a `ryuk` reaper container. The same ryuk id across checks confirms one
  continuous run, not a restart. No containers and no gate-log line means the run is dead.

## Memory

- The Docker Desktop VM (WSL 2) is capped in `.wslconfig` (reference project: 6 GB on 16 GB machines). The cap limits the VM only.
  See `windows-powershell.md` before changing it.
- A gate in flight left about 0.8 GB free: the WSL VM about 3.5 GB, `dotnet` about 1.3 GB, the test host about
  0.5 GB.

## Seeding

- The seed rebuilt the database from nothing. The same seed and anchor date gave the same data, checked at phase ends
  by seeding twice and diffing dumps, ignoring named time-recording columns.
- The seed refused to run unless the pinned dependency checkout was at the pinned commit. Fetching the commit isn't
  enough; it must be checked out. While an app gate or seed reads that checkout, keep it still: merge the dependency
  from a separate clone.
- Reseed before any test that reads the seeded database. A stale database fails in ways that look like regressions.
