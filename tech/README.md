# Technology notes

The core playbook is stack-neutral. These files hold what we learned about specific tools. Read a file only when your
stack uses that tool. Each one opens by saying when it applies, then gives the general pattern and the setting we
used on the reference project.

| File | Read it when your stack uses |
|---|---|
| [claude-code-harness.md](claude-code-harness.md) | Claude Code as the orchestrator, with subagents (always, for this playbook) |
| [windows-powershell.md](windows-powershell.md) | A Windows workstation, PowerShell 7 scripts, Git Bash beside them |
| [dotnet.md](dotnet.md) | .NET SDK builds and tests (MSBuild, Microsoft.Testing.Platform, xUnit) |
| [postgresql.md](postgresql.md) | PostgreSQL, Liquibase or other changelog migrations, schema-per-module ownership, plan tests |
| [docker-and-local-stack.md](docker-and-local-stack.md) | Docker Desktop, Compose dev stacks, Testcontainers, a secrets agent, local TLS |
| [playwright-e2e.md](playwright-e2e.md) | Playwright end-to-end tests against a seeded local stack |
| [gcp-demo.md](gcp-demo.md) | A shared demo hosted on a single cloud VM (we used a Google Cloud free trial) |

Figures and incidents from the reference project appear only as evidence. You don't need to know the project.
