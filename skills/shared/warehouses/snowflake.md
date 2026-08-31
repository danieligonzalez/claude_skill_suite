# Snowflake

Separate storage and compute. Queries run on a virtual warehouse you size and pay for by the second; data is stored in micro-partitions that the engine prunes when your filters let it.

**Identify:** `profiles.yml` `type: snowflake`; a `snowflake_warehouse` / `account` / `role` in the connection.

## What it costs

**Virtual-warehouse compute time, billed per second** (60-second minimum on resume), scaled by warehouse size (XS, S, M... each step is roughly double the credits per second). Storage is cheap and separate. So the cost of a query is roughly: warehouse size times how long it runs, and how long it runs is driven by how much data it has to scan after pruning. Three levers move the bill: prune more, cache more, and do not run a bigger warehouse than the query needs.

## Efficiency guardrails

- **Lean on the result cache.** An identical query text against unchanged data returns from cache for free and instantly (24-hour window, extended on reuse). Stable, re-run profiling and validation queries hit it; do not defeat it with a needless `current_timestamp()` or a changing comment.
- **Prune with filters on the clustering key.** Micro-partition pruning uses min/max metadata per column. A filter on a well-clustered column (often a date) skips partitions; a filter on an unclustered high-cardinality column scans everything. Filter on the clustering/date column whenever the question allows.
- **Never `select *` on wide or `variant` tables.** Columnar storage means unused columns are free to skip; `select *` gives that back and forces `variant` deserialization.
- **Right-size the warehouse.** Exploration and validation belong on an XS/S warehouse. A larger warehouse finishes faster but costs proportionally more per second, so it only pays off when the work is genuinely parallelizable. Let it auto-suspend.
- **Sample instead of scanning for shape checks.** `select ... from t sample (1000 rows)` or `sample (1 percent)` answers "what do these values look like" without a full scan.
- Avoid unnecessary `order by` (it forces a sort over the whole result) and self-cross-joins.
- Read `EXPLAIN` and the Query Profile when a query is slow: the partitions-scanned-vs-total ratio tells you whether pruning worked.

## Cheap probes (exploration and validation)

- Row counts and metadata come from `INFORMATION_SCHEMA` views and `SHOW TABLES` without scanning data.
- For grain and duplicate checks, filter to a recent partition window when the full history is not needed; state the window in the brief.
- Use `sample` for value-domain and null-shape peeks on large tables.

## Dialect and idiom notes

Snowflake and DuckDB agree on most of what the shapes use.

- `qualify` supported.
- `lag` / `lead` support `ignore nulls`.
- `is distinct from` supported.
- `::` cast and `cast()` both work.
- `date_trunc('month', d)`; `datediff('day', a, b)` (part-first) counts boundary crossings, same caution as DuckDB.
- `ilike` for case-insensitive match; `flatten` for `variant`/array; `sample (n rows | n percent)` for sampling.

## Incremental and materialization notes

- Incremental strategies: `merge` (default, needs `unique_key`), `delete+insert`, and `append`.
- Set the compute warehouse per model with the `snowflake_warehouse` config when a heavy fact should build on a bigger warehouse than the default.
- Cluster large, frequently-filtered tables with the `cluster_by` config on the model; it pays off only above tens of GB, so do not cluster small tables.
- `insert_overwrite` is not the idiomatic Snowflake strategy the way it is on BigQuery; prefer `merge`.
