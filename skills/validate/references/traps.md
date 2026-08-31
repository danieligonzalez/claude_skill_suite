# Traps checklist

One line yes or no per trap, against the SQL under validation. Dialect notes assume DuckDB here and Snowflake in production.

- **Coalesce on a flag.** `coalesce` is for missing values, not for combining conditions. A flag from `case when` is never null, so a coalesce fallback on it is dead code hiding a logic gap.
- **Filtered before aggregating.** For each `where` that runs before an aggregate: which aggregates needed the dropped rows? `count(*)` breaks even when `min` and `max` survive.
- **Ties in top N.** `rank() <= n` honors ties, `dense_rank() <= n` over-includes value tiers, `row_number()` picks arbitrarily. Push one tied example through before trusting the choice, and give `row_number()` a tiebreaker so it is deterministic.
- **`not in` with a nullable subquery.** One null in the subquery returns zero rows through three valued logic. Default to `not exists`.
- **Three valued logic elsewhere.** `!=` filters drop null rows, joins never match null to null, `count(col)` skips nulls, `case when x = null` never fires. For null safe comparison use `is distinct from`; DuckDB and Snowflake both support it.
- **Elapsed time from date_diff.** `date_diff` (DuckDB) and `datediff` (Snowflake) count boundary crossings: 23:59:59 to 00:00:01 is 1 day. When the question is elapsed time, difference in seconds and convert.
- **Propagating a group start forward.** `lead` looks forward; carrying a session or group start along its rows needs the running sum idiom, or `last_value ... ignore nulls` looking back. Both dialects support `ignore nulls` on `lag` and `lead` too, so the BigQuery workaround habit does not apply.
- **Merge key on an incremental model.** A merge on the wrong or non-unique key silently duplicates or overwrites. The idempotency check (build twice, counts equal) catches it.
- **Casts on raw varchar.** Raw columns are all varchar. A comparison or aggregate on an uncast column compares text: `'9' > '10'` is true. Every date and numeric use of a raw column casts first; staging exists so this happens exactly once.
