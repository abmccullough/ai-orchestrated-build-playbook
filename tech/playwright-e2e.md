# Playwright end-to-end tests

**Applies when your stack uses Playwright end-to-end tests against a seeded local stack.** The reference project ran Chromium and Edge
projects against the full edge stack (Caddy, API, worker, web) after a reseed, as an optional `-E2E` part of the merge
gate. That added several minutes on its own and made the full `-E2E` gate long.

## When to run it

The orchestrator decides per batch and writes the reason in the brief. Run end-to-end when the batch touched
something only a live stack exercises:
- screens (anything that renders);
- host composition, startup, configuration or authentication (what an in-process test host stubs);
- the dev stack (Compose, the edge proxy, the secrets setup);
- seeding of the profile the end-to-end tests read;
- text a screen renders that no typecheck sees: report labels, formats, data sources. Skipping it once left `main`
  red.

Otherwise the API contract snapshot plus the web typecheck against the regenerated client cover the contract. Phase
sweeps always run end-to-end.

## Spec changes are proved by the merge gate

- An implementer doesn't run its own end-to-end run just for edited specs. Its other lanes would repeat most of
  the gate's runtime. Two batches each paid that to find one locator.
- If the merge gate fails on a spec, prove the fix with one narrowed run of only the web and e2e lanes, classified
  against the failed gated SHA (a few minutes). That isn't a re-run to get past a failure, because it runs only
  what the fix touched.
- After a red gate whose other lanes passed, re-run just the missing lanes on the same SHA, and record both log
  lines.

## Locators

- **Every locator states `exact: true`.** A fix that skipped this was caught by the merge gate.
- **Never `.first()` or `nth()`.** Closed menus expose no links (not rendered, or hidden), or locators are scoped to
  the open menu by role.
- Each spec owns its data. One spec raced another spec's seed until it did.
- An identity used in a spec must have the right role in the end-to-end seed. One assertion failed because the chosen
  persona was a reader there.

## Infrastructure inside the e2e lane

- Everything before Playwright starts (npm ci, the browser install, the local CA, the seed, the stack coming up) is
  infrastructure. Retry it once, and keep its full output. A seed that fails once and passes on retry goes in the
  flake register with its error text.
- One end-to-end run per machine (see `docker-and-local-stack.md`). Never a seed beside an end-to-end gate.
- After an end-to-end run, a process can still hold the seed's files. A follow-on seed of another profile ran from a
  second fresh clone, not the gate's clone.
- `web/node_modules` older than the pulled code fails the edge build with "failed to resolve import." Run `npm ci`.
- An edge-only `net::ERR_NO_BUFFER_SPACE` was seen once and never again. Register it, and don't chase it.

## Component tests beside e2e (Vitest)

- Under load, a 1-second `findByRole` waiting on a lazy route timed out. Fix the wait, never a timeout: wait for the
  specific content you assert.
- **A race check:** an env switch (`APP_STUB_DELAY_MS=50`) delayed every stubbed API response. A screen test that
  fails only with the delay asserts before the result it waits for.
- A batch built on another unmerged batch runs the whole Vitest suite, not just its folders. A header test broke
  that only a reviewer's whole run found.
- Vitest with `pool: 'threads'` shares one V8 heap across workers. Its limit isn't system RAM. Check this before
  blaming the machine.

## Visual pass

Automated end-to-end tests don't replace a human-style pass. The reference project did a browser pass at phase ends, through the
owner's signed-in Chrome session. The owner valued it because it "finds things other tests can't see." Measure a CSS
finding in a real browser before fixing it. One suggested one-line fix pushed the list off the window.
