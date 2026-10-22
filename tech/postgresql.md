# PostgreSQL, migrations and plan tests

**Applies when your stack uses PostgreSQL**, especially with changelog migrations (the reference project used Liquibase formatted SQL),
per-module schemas and roles, and tests that assert query plans.

## One version, one floor

- **Test on exactly one version, the enterprise minimum** (reference project: `postgres:15.1` everywhere: Testcontainers, the dev
  stack and both gates). A second version doubled the gate cost for a compatibility risk the owner didn't weigh
  highly. Ask before you add one back.
- **A newer local server proves nothing.** The reference project stopped the native PostgreSQL 18 service entirely. Nothing used it.
- **Enforce the SQL floor with a static scan** of every `.sql` file (for example, no `MERGE` below 15). The scan
  misses ORM-generated SQL, so call out role, grant, `SECURITY DEFINER` and `search_path` changes as unchecked
  compatibility risk in checkpoint reports.

## Migrations

- **Every schema change is a new changeset file**, `db/changelog/<module>/NNNN-<description>.sql`, included from the
  root. No ORM migrations, no `EnsureCreated`, no manual DDL.
- **Never edit a merged changeset.** The gate compares every versioned changeset on `origin/main` byte for byte with
  the branch (`runOnChange` files excepted). A branch missing a changeset that `main` has fails here in under a minute.
  That's not a defect: merge `main` forward.
- **`liquibase validate` runs in the gate** against an ephemeral bootstrapped container, using a pinned Liquibase
  image (no Java install).
- **A second repository's changelog is pinned by commit** (for example, a `pipeline.lock` file). The app exports it from that commit
  before any integration test. A fresh clone must run that export first, or most integration tests fail in setup and
  say nothing about the batch. Every merge to the dependency's `main` bumps the pin straight away. A stale pin fails
  every end-to-end gate at reseed.
- **One-shot migration containers re-run on restart.** See `docker-and-local-stack.md`. Restarting a stopped stack
  with `docker start` on every container applied a branch's `DROP TABLE` conversion early.

## Per-module ownership

- **Each module owns one schema, has its own login role, and is that schema's only writer.** Other modules read through
  granted read views, and grants live in changesets.
- **A boundary violation shows up as `42501 permission denied for schema`.** Once it surfaced on a rare path, 2 of
  40 scheduled runs under load. It looked like a race but was a module reading another's table directly. Read the
  SQL in the error before calling it infrastructure.
- **A batch that grants itself another module's read view runs that module's schema tests.** The owner's tests may
  assert who holds its views, and a new grant makes them stale. Neither implementer nor reviewer caught one case. The
  merge gate did.
- **A test that narrows another module's view renames the real view aside and back.** It never drops and re-creates
  it: the drop fails with `2BP01` once another view depends on it, and a re-create loses grants.
- A reporting role was granted dataset views and nothing else, kept in a separate schema, so the grant could be
  exact.

## Test isolation with a shared database

- **No test owns a shared table.** A global "published dates" table and the watermark derived from it move forward
  as other tests publish (one test saw a date in the year 6397). Anchor dates on the live watermark, and assert only
  what your own rows make true.
- **Use a shared date counter as given, never shifted.** `counter.AddDays(1)` is exactly the next caller's date. Need
  two days? Make two counter calls. A hand-inserted row in a table a counter-dated job versions takes its dates from
  the counter too, or the next run closes it before it opens (`valid_to < valid_from`) and every later caller fails.
- **A test of an organization-wide job asserts only on its own project's rows**, never on run totals, and uses a
  date no other test uses.
- **A status computed "as of" a resolved date dates its data from that date, not from today.**

## Plan tests

The reference project's plan tests asserted that hot reads stay bounded. They caused more flakes than anything else until these rules
held:

- **Seed your own volume** (thousands of rows), inside the transaction, and bind your own values. Never assert a plan
  over tables the test didn't seed, and never use `enable_seqscan = off` over a near-empty table.
- **Seed every table the plan joins**, not only the one you assert on. A planner flipped on two tables that held only
  other tests' few rows.
- **Seed the rows the statement doesn't ask for**, in production's ratio. An index-condition assertion holds only when
  the asked keys are a small share of the table.
- **After seeding, analyzing and rolling back, `VACUUM` the tables.** `ANALYZE` writes `reltuples` and `relpages` in
  place, and only `pg_statistic` rolls back, so the next class plans against phantom rows. `VACUUM` doesn't shrink a
  bloated index, so seed enough that plan choice doesn't depend on leftovers.
- **`ANALYZE` partitioned parents as well as their leaves.** Joins over a partitioned table use the parent's inherited
  statistics. Without them the planner falls back to a default selectivity, and leftover bloat picks the winner.
- **Assert the defect, not one plan shape:** a table read once per item, a whole-set read at each step, the probe loop
  count, rows into the expensive step. Assert the index *condition*, not just that an index is named. A plan that
  depends on a guessed array length or a neighbor's rows fails by neighbor.
- **Buffer allowances must not include per-row scans of tables other classes churn.** Dead rows wait for autovacuum,
  which runs later in a full run than in a folder run. Tie the allowance to the table's page count.
- **When two indexes can each answer a read, seed the data under which the asserted one is genuinely better.**

## Planner patterns that cost us

Rule: build for 100x to 1000x local data, and fix a cost that grows with data when you see it. Patterns found and
fixed:

- **N+1:** per-row lookups on list pages. Batch by id (watch request-line limits: the reference project used 150 ids from the browser
  and 500 from the API).
- **Recompute everything:** a one-row change triggering a whole-table rebuild. Refresh only what changed.
- **Aggregate in SQL**, not in the API: about 9,350 rows fetched per call and summed in code.
- **Index every hot filter column**, on the partitioned parent, so rebuilds keep it.
- **Store history as validity ranges, not a row per date:** 2.44 GB became 28 MB. Join per-date history by range
  directly (`valid_from <= d < valid_to`), never through an expanded per-date view. The planner can't estimate the
  expansion.
- **Bound partitioned history reads** with `valid_to > $earliest` passed as a parameter, so partitions are pruned at
  planning. A bound computed inside the statement isn't. Give keys as `= ANY(ARRAY(SELECT ...))` so each partition
  probes by index, but measure first: when the keys are a large share of the table, a bounded sequential read was
  twice as fast.
- **`LEFT JOIN ... WHERE x.id IS NULL` misestimates after writes.** Once dismiss/restore cycles left only cleared
  rows, the estimate fell to 1 (actual 12,723) and every join above became a per-row loop (183 ms to 3.7 s). Use
  `NOT EXISTS`. Plan tests seed tables in their post-write state.
- **CTEs:** a single-reference CTE joined to dates on a range was better `MATERIALIZED`. But a CTE has no
  statistics, so a grouped subquery beat a materialized CTE when the join needed estimates. Measure both. Select only
  the columns you need: a wide column carried through a multiply-referenced CTE spilled to temp.
- **A scope filter applied above a big join does no good.** An OR scope on one side's columns was applied last, so
  project-scoped reads cost as much as organization-wide ones. Fence the scope onto the driving side first.
- **A `WHERE` after a view join may not cut the work.** The same predicate in a CTE read first cut rows into the view
  from 20,300 to 4,404.
- **Compute per-row constants once** (a watermark or publication date) in a materialized CTE read through a scalar
  subquery.

## Connections and statistics

- **Test container `max_connections=500`**, with the peak printed in every gate. A `53300` means a new leak.
- **Don't hold about 2,000 connections through Docker Desktop's port forwarder** in a probe. Beyond that it closes new
  connections to every container, breaking other processes' tests.
- Statistics functions reveal row counts outside a user's access. The reference project's owner ruled that counts aren't confidential.
  Decide this explicitly for your organization.
