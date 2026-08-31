# BigQuery

Serverless and columnar. There is no warehouse to size; you are billed for the **bytes a query scans**, and the whole optimization game is scanning fewer bytes.

**Identify:** `profiles.yml` `type: bigquery`; a `project` / `dataset` and a keyfile or oauth in the connection.

## What it costs

**On-demand pricing bills bytes scanned**, not rows returned and not wall-clock time. This has one consequence that surprises people from every other warehouse:

- **`LIMIT` does not reduce cost.** `select * from big_table limit 10` scans every byte of every column referenced, then throws away all but 10 rows. The bill is the same as without the `LIMIT`. Use the free table preview or `INFORMATION_SCHEMA` for a peek, never `select * limit n`.

Bytes scanned is driven by two things: which **columns** you touch (columnar storage skips the rest) and which **partitions** you touch (a partition filter prunes the rest). Miss either and you scan the whole table.

## Efficiency guardrails

- **Always filter on the partition column.** Tables are usually partitioned by a date (or ingestion time, `_PARTITIONTIME`). A query without a partition filter scans all history. Many tables set `require_partition_filter`, which rejects the query outright; treat that as the norm, not an error.
- **Select only the columns you need.** Never `select *`. Use `select * except(heavy_col)` to drop known-large columns when you truly need the rest.
- **Cluster inside partitions** for high-cardinality filter columns; clustering prunes blocks within a partition.
- **Dry-run to see the bill before you pay it.** A dry run returns the bytes a query would scan without running it (`bq query --dry_run`, or the estimate in the console/editor). Estimate before running anything against a large table. Set `maximum_bytes_billed` as a guardrail so a mistake fails instead of billing.
- **Use approximate aggregates** when exact is not required: `approx_count_distinct(x)` scans the same bytes but is far cheaper in slot time than `count(distinct x)` on high-cardinality columns.
- **Sample with `tablesample system (n percent)`**, which reads a subset of blocks, for shape checks.

## Cheap probes (exploration and validation)

Prefer metadata over scans wherever possible:

- Row counts and table size come from `INFORMATION_SCHEMA.PARTITIONS` / `__TABLES__` (`row_count`, `size_bytes`) with zero scan. Use these for the grain-proof row count instead of `count(*)` on a huge table.
- When you must scan (distinct-key counts, null profiles, domains), scope to one partition or a `tablesample` and say so in the brief. Dry-run first.
- Table preview in the console is free; use it for the 5-row shape peek instead of a `limit` query.

## Dialect and idiom notes

BigQuery (GoogleSQL) differs from the DuckDB shapes in several places; translate before running.

- `qualify` supported (requires a `where`/`group by` or `window` in the same query).
- **`date_diff` and `date_trunc` take arguments in a different order:** `date_diff(end, start, day)` and `date_trunc(date, month)`. Do not copy DuckDB's `datediff('day', a, b)` verbatim.
- Cast with `cast(x as type)` or `safe_cast(x as type)`; there is no `::` operator. `safe_cast` returns null instead of erroring on bad input, which is the right default when profiling raw strings.
- `is distinct from` supported. `countif(cond)` replaces `sum(case when cond then 1 else 0 end)`. `safe.` prefix guards a function against errors.
- Strings, arrays, and structs are first-class; `select * except(col)` and `select * replace(...)` trim wide rows.

## Incremental and materialization notes

- **`insert_overwrite` on partitions is the efficient incremental strategy.** It replaces whole partitions and only scans the partitions in play, which is both cheap and idempotent. Prefer it over `merge` for partitioned facts.
- Set `partition_by` (with `granularity`) and `cluster_by` in the model config; add `require_partition_filter` on large facts so downstream queries cannot forget the filter.
- `merge` is available for non-partitioned or key-based upserts, but it scans more; reach for it only when partition-overwrite does not fit.
