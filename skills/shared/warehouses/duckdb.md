# DuckDB

The suite's reference adapter. Local, single node, columnar, embedded in the process. The [query shapes](../query-shapes.md) are written in DuckDB idiom, so on DuckDB they run as written.

**Identify:** `profiles.yml` `type: duckdb`; a local `.duckdb` file (or `:memory:`). Usually paired with local dbt Core, not dbt Platform.

## What it costs

Almost nothing in dollars. DuckDB runs on the local machine against a local file, so there is no per-query bill and no warehouse to right-size. The two real costs are **memory** (a large aggregation or join can exhaust RAM and spill) and **the single-writer lock** (below). Optimize for correctness and for not fighting the lock, not for scan cost.

## Efficiency guardrails

- The cost model is generous, but the discipline is not: aggregates only, no row dumps. A profiling query that returns one row is free; a `select *` that streams a million rows into context is expensive in tokens even when it is cheap in compute.
- Prefer `count(*)` and grouped aggregates. Use `using sample 5 rows` or `limit 5` when the shape of a value matters.
- DuckDB reads CSV and Parquet in place with `read_csv_auto('/abs/path')` and `read_parquet(...)`; you can profile a file before it is ever loaded. Absolute paths only.

## The single-writer lock

DuckDB allows one writer on the database file at a time. In a project that also runs the DuckDB UI, the UI holds the lock while its database is attached, so dbt must hand the lock around. That is what the `dbtw` wrapper does: it detaches the database from a running UI over the UI server's local HTTP endpoint, runs dbt, and re-attaches. With no UI running it is a plain `dbt` call, so `dbtw` is always the right way to run dbt in such a project.

- `dbtw` does not cover non-dbt writers (an ingest script, a Power User query preview, the `edr` CLI). For those, detach first in the UI: `USE memory; DETACH warehouse;` then re-attach after.
- The UI must start bare (`duckdb -ui`), never with the database file as an argument; a main database can never be detached.
- Do not kill lock holders as a first move. `profiles.yml` with `keep_open: false` plus connect retries lets terminal dbt, the Power User extension, and scripts hand the lock around on their own.

If a local DuckDB project ships no such wrapper, plain `dbt` is correct and the lock notes do not apply.

## Cheap probes (exploration and validation)

Everything is cheap, so the probe set is the full battery in [query-shapes.md](../query-shapes.md) with no cost trimming. Run the grain proof, the duplicate and orphan checks, the null profile, and the date coverage as written.

## Dialect and idiom notes

DuckDB is the reference dialect; these are the features the shapes rely on.

- `qualify` supported.
- `lag` / `lead` support `ignore nulls`.
- `is distinct from` for null-safe comparison.
- `::` cast and `cast(... as ...)` both work. Raw columns land as `varchar`; cast before date or numeric work.
- `date_trunc('month', d)`, `datediff('day', a, b)`. `datediff` counts boundary crossings (23:59:59 to 00:00:01 is 1 day); for elapsed time, difference in seconds and convert.
- `list`, `struct`, and `read_csv_auto` / `read_parquet` are DuckDB extensions with no cross-warehouse equivalent.

## Incremental and materialization notes

- Incremental strategies: `delete+insert` (default) and `append`; `unique_key` drives the merge.
- Datasets in a local DuckDB project are often static, so an incremental model never meets a real second batch. The build-twice-then-full-refresh parity checks in the build-model template are the only evidence the incremental path works; they are mandatory, not optional.
- No clustering or partition keys to tune; a single-node columnar scan does not reward them.
