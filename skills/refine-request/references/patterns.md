# Pattern catalog

Roughly 90 percent of analytics SQL maps to one of these. Recognition is the speed: a named pattern is a query already written. For sequence patterns the mantra is: flag the starts, derive the rest by grouping. Ends are derivable; starts are the signal.

| Pattern | Trigger phrase | Idiom |
|---|---|---|
| Aggregate plus filter on it | "categories with more than..." | `group by` plus `having` |
| Top N per group | "top 3 per...", "highest in each..." | `rank()` or `row_number()` plus `qualify` |
| Latest record per key | "most recent...", "current state of..." | `row_number() = 1` plus `qualify` |
| Gaps and islands | "consecutive...", "sessions...", "streaks..." | flag the starts plus running `sum` |
| Anti-join | "never...", "without any...", "missing from..." | `not exists` |
| Prior or next row comparison | "change from previous...", "growth...", "time between..." | `lag` / `lead` |
| Running or rolling | "cumulative...", "trailing 7 day..." | window frame (`rows between ...`) |
| Cohort or offset | "retention...", "months since...", "by cohort..." | anchor table plus date offset plus join back; two required considerations travel with this pattern: maturity (a cohort younger than the window has a null rate, never a low one) and left-censoring (the first observed window mixes new entities with pre-existing ones; flag it) |
| Ratio or rate metric | "rate...", "percent of...", "conversion..." | numerator over the eligible population only, never the raw population (immature or ineligible rows leave the denominator); store the numerator and denominator beside the rate so rollups re-divide instead of averaging averages; an empty denominator is null, not zero |
| Hierarchy of unknown depth | "org chart...", "category tree...", "manager's manager..." | `with recursive` (base case plus recursive join, depth cap against cycles); known small depth uses self joins instead |
| SCD Type 2, build | "turn this change log or these snapshots into history..." | change log: `lead(changed_at)` as `valid_to`; daily snapshots: hash tracked columns plus gaps and islands; the production answer is a dbt snapshot |
| SCD Type 2, query | "as of the order date...", "customer as they were..." | point in time range join: `>= valid_from and < coalesce(valid_to, '9999-12-31')`, half open intervals; current state questions use `is_current` |

## Dialect notes

This repo runs DuckDB; production warehouses are often Snowflake. The two agree on the points below, which differ from BigQuery habits:

- `qualify` works in both.
- `lag` and `lead` support `ignore nulls` in both. The BigQuery workaround through `last_value` is unnecessary here.
- `is distinct from` works in both. No coalesce null-equality workaround needed.
- `date_diff` (DuckDB) and `datediff` (Snowflake) count boundary crossings, not elapsed intervals: 23:59:59 to 00:00:01 is 1 day. When the question is elapsed time, compare in seconds and convert.
