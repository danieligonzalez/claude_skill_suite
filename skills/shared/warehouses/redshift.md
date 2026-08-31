# Redshift

A clustered, node-based MPP warehouse (provisioned nodes or serverless RPUs). Performance is decided less by query text than by physical design: how data is **distributed** across nodes and **sorted** within them.

**Identify:** `profiles.yml` `type: redshift`; a `host` ending in `redshift.amazonaws.com` or a serverless workgroup.

## What it costs

**Provisioned node-hours, or serverless RPU-hours.** Either way the lever is how much data each query has to scan and how much it has to shuffle between nodes. Two physical-design choices dominate:

- **Distribution (DISTKEY / DISTSTYLE).** Rows are spread across node slices. When a join's two sides share a distribution key, the join is local; when they do not, Redshift redistributes one side across the network mid-query, which is the single biggest hidden cost. Join on the distribution key where you can.
- **Sort keys (SORTKEY).** Zone maps store min/max per block for the sort key, so a filter on the sort key skips blocks. A filter on a non-sort column scans the table.

## Efficiency guardrails

- **Filter on the sort key** to get zone-map pruning; a date sort key plus a date filter is the common win.
- **Join on the distribution key** to avoid redistribution. If you see a slow join, check whether the two sides are distributed compatibly.
- **Avoid `DISTSTYLE ALL` on large tables** (it copies the whole table to every node) and avoid `select *` (Redshift is columnar; unused columns are free to skip).
- **Keep tables maintained.** Stale statistics and unvacuumed deletes slow scans; `ANALYZE` and `VACUUM` (or rely on the automatic ones) matter more here than on serverless warehouses.
- **`LIMIT` still scans** before it truncates, so it does not save a full-table cost the way people expect; scope with a sort-key filter instead.
- **Unload big extracts to S3** with `UNLOAD` rather than streaming a large result through the client.
- Read `EXPLAIN`; `DS_BCAST_INNER` / `DS_DIST_BOTH` in the plan flag the expensive redistribution steps.

## Cheap probes (exploration and validation)

- Table size and row counts come from `SVV_TABLE_INFO` and `PG_CLASS` without scanning data; use them for the grain-proof count on large tables.
- Scope grain, duplicate, and null checks to a sort-key range when full history is not needed, and state the window.
- Use `APPROXIMATE COUNT(DISTINCT x)` for high-cardinality distinct counts.

## Dialect and idiom notes

Redshift is the adapter that diverges most from the DuckDB shapes; check here before running any window-function shape.

- **`qualify` is NOT supported.** Rewrite `qualify row_number() over (...) = 1` as a CTE that computes the `row_number()` and an outer `where rn = 1`. Every top-N-per-group and latest-record shape needs this translation.
- `is distinct from` supported.
- Cast with `cast()` or `::`. `date_trunc('month', d)`; `datediff(day, a, b)` (part-first, unquoted part) counts boundary crossings.
- `lag` / `lead` support `ignore nulls`, but confirm on your cluster version if a shape depends on it.
- `listagg` for string aggregation; `approximate count(distinct ...)` for cheap cardinality.

## Incremental and materialization notes

- Incremental strategies: `append`, `delete+insert`, and `merge` (native `MERGE` is supported on current Redshift).
- Set `dist` and `sort` in the model config (`dist='customer_id'`, `sort='payment_date'`); these are the highest-leverage tuning knobs and belong on every large fact.
- Choose the distribution key to match the table's most common join key, and the sort key to match its most common range filter.
- Serverless removes the node-sizing decision but not the dist/sort decision; keys still govern scan and shuffle cost.
