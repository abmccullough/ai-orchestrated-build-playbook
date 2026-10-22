# .NET

**Applies when your stack uses the .NET SDK** (reference project: SDK 10.0.4xx, Microsoft.Testing.Platform as the test runner,
xUnit, Testcontainers, a modular monolith).

## Gate shape (pattern)

The reference project's `verify.ps1` dotnet lane, in order:
1. Check the SDK band that `global.json` pins.
2. Check that Docker is available, before restore, so failures come fast and clear.
3. `dotnet restore --locked-mode` (a stale lock file fails the gate).
4. `dotnet tool restore`.
5. Build in Release with warnings as errors (`Directory.Build.props`).
6. Database steps: export the dependency's changelog, `liquibase validate`, changelog immutability, a static SQL scan.
7. Unit, Integration and Architecture tests with coverage.
8. Merged coverage at 80 % or more of `src`.

Lanes are chosen by diff. Any uncertainty (no merge base, a dirty tree, an empty diff, a change to the gate itself)
runs every lane: a bug in lane selection should cost time, never coverage.

## Build servers and memory

- **MSBuild node reuse leaves idle `dotnet.exe` workers** (about 1 GB per build set) running after builds, gates and
  seeds. On a 16 GB machine this starved Windows and caused out-of-memory failures that were misdiagnosed for a day.
- Fix: set `MSBUILDDISABLENODEREUSE=1` for every agent shell (Claude Code project `settings.json` `env`) and in the
  gate script. For leftovers from older sessions, run `dotnet build-server shutdown` when nothing is building. It is
  graceful and also stops the Roslyn server. Never kill the workers by name.
- The test platform's end-of-run summary printer threw `OutOfMemoryException` at about 1,000 integration results on a
  machine with a small fixed page file. Every test had passed. Check the page file and commit limit before blaming
  the suite.

## Microsoft.Testing.Platform traps

- **Run `dotnet test` from inside the clone.** `global.json`'s `test.runner: Microsoft.Testing.Platform` is found
  from the working directory. Outside the clone, `dotnet` falls back to VSTest and rejects `--project` with
  `MSB1001: Unknown switch`. It looks like a broken SDK, but it's only the folder.
- **`dotnet test -- -namespace ...` silently runs zero tests** with this runner. Use `--filter-class` or
  `--filter-method`. A trailing `*` works, but a `*` in the middle matches nothing, so list the classes. A filtered
  run that reports 0 tests ran nothing.
- Targeted runs: `dotnet test --project tests/Integration/Tests.Integration.csproj -c Release --no-build --output
  Detailed --filter-class <Full.Class.Name>`. Keep the output in a file named in the progress log. A flake once
  couldn't be identified because its output was dropped.

## Test isolation

- **Each integration folder is one xUnit collection with its own cloned database** from a template. Collections run in
  parallel, and classes within one run in order.
- **Class order differs between build paths** (the checkout, a scratch clone, each gate clone), but is stable within
  one. A test that depends on rows other classes commit can pass every targeted run and fail in one gate. Reproduce
  by running the folder from a second path.
- **Dispose every `DbContext` a test builds** (`await using`). Undisposed fixture contexts leaked pooled connections
  past PostgreSQL's default 100 (`53300`). The reference project raised the test container to `max_connections=500`, printed the peak
  in every gate, and treats a peak above 400 as a follow-up.
- **A test that alters an object a host's startup check reads starts the host first.** Otherwise it passes when
  another test happened to start the host and fails when it runs first in its class. Its fail check is the test run
  alone.
- **The unit project's gate checks read configuration and test-support files as text.** A change there runs the whole
  unit project, even when "nothing it compiles changed."
- Slow tests that guard rare, stable failures (startup refusal, about 41 s) carry a `SignOff` trait. Normal gates
  leave them out, and phase sweeps and batches touching that code include them.

## Fail checks

- **Restore a probed file with `git checkout -- <file>`, not a copy.** A copy keeps the backup's old modification
  time, so the next incremental build skips the compile and runs the probed code: a false fail or pass. If a copy was
  used, touch the file.
- Batch all fail-check probes into one scripted pass: break, run the filter, restore, confirm clean, for every probe.

## Modules

The reference project enforced module boundaries with architecture tests:
- Each module is `<Module>.Contracts` plus `<Module>`, and references another module only through `.Contracts`.
- Hosts hold no business logic.
- Each module derives its own `DbContext` from a shared base, registered through one helper that sets the provider,
  the pool cap and a rotating credential source.

See `postgresql.md` for the database side.
