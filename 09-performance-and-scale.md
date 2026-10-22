# 9. Performance and scale

Generalized from the reference project's performance-lessons file, started at the owner's request "to stop repeating the same performance
mistakes". Keep your own as a living log: each pattern with what it looks like, why it costs what it costs, the general
rule, and every instance found with its status. Consult it when briefing or reviewing anything that touches a list, a
job or a query. Database-specific detail (planner behavior, index types, statistics quirks) is in the tech notes in
[`tech/`](tech/).

## 9.1 The standing rule: build for 100x–1000x local

A cost that **grows with data** and has a **known fix** is fixed when it is seen, even when today's numbers are well
inside target. In adjudication it counts as a fix-now reason. "Close" and "watch" are only for costs with no growth
term (constant round trips), and those still need numbers.

*Why:* development data is small and margins measured there say little about production. The fixes are cheapest before
the pattern is copied across screens.

## 9.2 Recurring patterns

| # | Pattern | What it looks like | General rule |
|---|---|---|---|
| 1 | **One row at a time (N+1)** | A list resolves a related field (owner name, linked item) with one query or HTTP request per row | One batched read per page. A list column that can only be filled by reaching past the row needs a reason beyond completeness; per-record fan-out belongs on the record page. A design that demands such a column is a design defect to raise |
| 2 | **Recomputing from scratch** | A job rebuilds a whole derived table on every single-row change; an API fetches raw rows and aggregates in code | Touch only affected rows; aggregate in the database; ask what it costs at 10x rows |
| 3 | **Uncompressed responses** | Large bundles or JSON served without compression | Compression on at the edge; re-check on every edge config change |
| 4 | **Unbounded fan-out per page** | Enrichment runs for the whole filtered set before the page slice is taken | Slice first, enrich after; cap per-group results and offer "show all" paging |
| 5 | **Hot lookup with no index** | A read filters or joins on an unindexed column | Every column a hot read filters or joins on has an index, placed where rebuilds and partitioning keep it; reviewers check the plan for new filtered reads on large tables |
| 6 | **A row per date instead of a range** | History stored as one row per item per day even when unchanged | Store validity ranges; a per-date shape is a view; join history by range directly, not through an expanded view the planner cannot estimate |
| 7 | **Unbounded join to history** | A join to partitioned history with no time bound | Bound history reads to the window with a bound **parameter** (so partitions prune), and pass keys in a form the planner can probe by index; but measure first: when the keys are a large share of the table, a bounded sequential read wins |
| 8 | **Re-evaluating a constant per row** | A correlated subquery or per-row probe recomputes a value that is the same for every row | Compute constants once; read each table once; write membership tests so they can hash |

Lessons that cut across patterns:

- **Measure the rows into the expensive step, not only the time.** A filter added after a join may change nothing if
  the planner applies it last.
- **Estimates break after writes.** A plan that is fine on fresh data can flip to per-row scans once ordinary
  edit cycles change the data's shape; plan tests seed tables in their post-write state.
- **Counter-cases are recorded too.** When the "fix" was measured slower (keying a read by a large key set; scoping an
  organization-wide read that is cheaper whole), record the measurement and the ceiling at which to revisit.

## 9.3 Guarding with tests

- **Plan tests** assert the defect itself (a table read once per item, a loop count, rows into a step), not one join
  shape among several good plans, and not buffers or time, which vary by machine.
- They seed their own volume, including the rows the statement does **not** ask for, in production's ratio, and refresh
  statistics on every table the plan joins. Otherwise the planner is right to read small tables whole, and the result
  depends on what earlier tests left behind.
- An equivalence test (stored equals live, for every path that writes) catches correctness gaps in incremental
  refreshes, which is where narrowing a recompute usually breaks.

## 9.4 Measuring at scale

- **Scale profiles** of the seed data: `standard` for tests, a mid scale, and a full scale matching the design's sizing
  basis. The full-scale seed exists to be measured: no functional tests or gates against it; its checks are that it
  completes within its ceilings and its row counts match. The code that builds it is still checked normally (smaller
  profiles' output unchanged).
- **Measurement campaigns** are their own batches (the reference project used an `m` suffix): a brief naming the scenarios, the observation
  scripts (slow statements, sequential scans on large tables, table sizes, connection peaks), the limits being tested,
  and the output folder. Results are a report, and every scaling cost found becomes a fix row with its numbers.
- **A second workstation** is ideal for long measurement or soak runs, so the main machine stays free for gates. Keep
  the two coordinated through files in git (brief, measurement log, results folder) plus short messages; record clock
  times for anything joined across machines.
- **Record every measurement**, even when the verdict is "fine": the next person measuring the same thing should not
  have to rediscover it.

## 9.5 Where to look in review

- List endpoints: any per-row loop, per-row request from the client, or singular fetch inside a loop where a batch
  form exists.
- Jobs: any full-table write triggered by a single-row change.
- Reads: any aggregation in application code; any join to history without a time bound; any new filtered read on a
  large table without a plan check.
- Storage: any table whose row count is about items x dates.
- Edge config: compression still on.
