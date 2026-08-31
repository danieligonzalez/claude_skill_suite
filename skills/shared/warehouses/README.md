# Warehouse guardrails

The skills in this suite write and run SQL. What counts as an efficient, cheap query is not the same on every warehouse: BigQuery bills bytes scanned, Snowflake bills warehouse seconds, Redshift rewards the right dist and sort keys, DuckDB is local and nearly free. A query that is fine on one is expensive on another. So before any skill writes or runs warehouse-touching SQL, it does two things: identify the warehouse, then read that warehouse's file below and follow its guardrails.

This is a hard step, not a nicety. explore-data, validate, quick-query, and build-model all gate on it (each names the gate in its own SKILL.md). Skipping it is how a profiling battery that costs nothing on DuckDB becomes a full-table scan that bills for terabytes on BigQuery.

## Identify the warehouse

Check in this order, first answer wins:

1. **dbt profile.** The adapter type in `profiles.yml` (or the dbt Platform connection) is the ground truth: `type: duckdb | snowflake | bigquery | redshift | databricks | spark`. `dbt_project.yml` names the profile; the profile names the type. On dbt Platform, the connection's adapter is available through the MCP (see [../runtime.md](../runtime.md)).
2. **The map.** [../warehouse-map.md](../warehouse-map.md) states the project's warehouse at the top when it has been filled in.
3. **Ask.** If none of the above answers, ask the user one sentence: which warehouse is this. Never guess from schema names or SQL style; a wrong guess sends every query down the wrong cost model.

Say the identified warehouse out loud once when a stakeholder is watching, the same way the runtime and the lane get said: it shows the cost model was chosen, not defaulted.

## The files

| Warehouse | File | Billing model in one line |
|---|---|---|
| DuckDB | [duckdb.md](duckdb.md) | Local single node. Nearly free; the cost is the single-writer lock, not compute. |
| Snowflake | [snowflake.md](snowflake.md) | Virtual-warehouse seconds. Prune with clustering, lean on the result cache, right-size the warehouse. |
| BigQuery | [bigquery.md](bigquery.md) | Bytes scanned. Partition-filter and select columns; `LIMIT` does not reduce cost. |
| Redshift | [redshift.md](redshift.md) | Cluster or RPU hours. Dist and sort keys decide everything; joins that redistribute are the tax. |
| Databricks | [databricks.md](databricks.md) | DBUs on the SQL warehouse. Delta file pruning via partition and Z-order or liquid clustering. |

Adding another adapter (Postgres, Trino, Athena, ClickHouse): copy the shape of an existing file, name it `<type>.md`, add a row here. The shape is fixed on purpose so any file scans the same way: identify, cost model, guardrails, cheap probes, dialect notes, incremental notes.

## Dialect: the shapes are written for DuckDB

The canonical query shapes in [../query-shapes.md](../query-shapes.md) are written in DuckDB idiom because that is the suite's reference adapter. Every warehouse file has a **Dialect and idiom notes** section listing where that warehouse differs (cast syntax, `qualify` support, `date_diff` argument order, null-safe comparison, sampling). Translate through that section before running a shape on another warehouse. The two differences that bite most often:

- **`qualify` is not universal.** Snowflake, BigQuery, and Databricks support it; Redshift does not (emulate with a CTE and a `where row_number = 1`).
- **`date_diff` argument order and semantics differ** across adapters, and most count boundary crossings rather than elapsed time. Check the file before trusting an interval.
