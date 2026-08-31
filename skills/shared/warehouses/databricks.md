# Databricks

A lakehouse: SQL over Delta Lake tables, run on a SQL warehouse (or cluster), often with the Photon engine. Data lives in Parquet files with Delta transaction logs and column statistics; performance comes from letting the engine skip files.

**Identify:** `profiles.yml` `type: databricks` (or `spark`); a `host` on `databricks.com`, a `http_path`, and usually a Unity Catalog `catalog`.

## What it costs

**DBUs, billed for the compute the SQL warehouse or cluster uses while it runs**, scaled by warehouse size and by whether Photon is on. As with Snowflake, cost is roughly warehouse size times run time, and run time is driven by how many Delta files the query has to open. The optimization game is **file pruning**: read fewer files.

## Efficiency guardrails

- **Prune with partition and clustering.** Filter on the partition column, and cluster high-cardinality filter columns with `Z-ORDER` (or liquid clustering on newer tables) so data skipping can drop files by their min/max stats.
- **Keep files healthy.** `OPTIMIZE` compacts small files (the small-file problem kills scan performance) and `VACUUM` removes tombstoned ones. A table written incrementally in many small batches needs periodic `OPTIMIZE`.
- **Never `select *`** on wide Delta tables; columnar skipping only helps if you name the columns.
- **Let the serverless SQL warehouse auto-stop**, and run exploration on a small warehouse. Photon speeds scans but is billed at a higher DBU rate, so it pays off on heavy scans, not tiny probes.
- **Sample with `tablesample`** for value-shape checks instead of scanning the whole table.
- Read the query profile / `EXPLAIN`; the files-pruned-vs-read number tells you whether skipping worked.

## Cheap probes (exploration and validation)

- `DESCRIBE DETAIL <table>` returns size, file count, and partition info with no scan; use it for the size read before a big `count(*)`.
- Under Unity Catalog, `information_schema` views give columns, tables, and constraints without scanning data.
- Scope checks to a partition window when full history is not needed, and state the window in the brief.

## Dialect and idiom notes

Databricks SQL is ANSI-leaning and close to the DuckDB shapes.

- `qualify` supported.
- `is distinct from` supported; `<=>` is the null-safe equality operator.
- Cast with `cast()` or `::`. `date_trunc('month', d)`; `datediff(day, a, b)` and `date_diff(...)` both exist, with the boundary-crossing caution.
- `lag` / `lead` support `ignore nulls`. `select * except(col)` trims wide rows. Higher-order functions (`transform`, `filter`, `aggregate`) operate on array columns.

## Incremental and materialization notes

- Incremental strategies: `merge` (default on Delta, needs `unique_key`), `insert_overwrite` (partition replace), and `append`. `replace_where` targets a predicate for partial overwrites.
- Set `partition_by` in the model config; for high-cardinality access prefer `liquid_clustering` (or a post-hook `OPTIMIZE ... ZORDER BY`) over partitioning on a high-cardinality column, which creates too many small files.
- `file_format: delta` is the default and the one that carries the statistics data-skipping relies on.
- Schedule `OPTIMIZE`/`VACUUM` for incrementally-built tables; without it, file count grows and every downstream scan slows.
