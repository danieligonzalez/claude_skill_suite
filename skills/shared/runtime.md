# Runtime: dbt Core vs dbt Platform (Cloud)

Which tools a skill reaches for depends on where the project runs. There are two runtimes, and they change how models are discovered, how queries run, and how builds happen. Identify the runtime once at the start of a session, the same way the warehouse gets identified, and route every dbt action through the matching column below.

The rule in one line: **on dbt Platform (Cloud), the dbt MCP server is the default; do not grep the whole repo to answer a question the discovery API already knows.** On local dbt Core, the CLI plus the filesystem is the default.

## Identify the runtime

Check in this order, first answer wins:

1. **The dbt MCP server is connected.** If dbt MCP tools are available in the session (names like `get_all_models`, `execute_sql`, `get_lineage`), this is dbt Platform and those tools are the default path. This is the strongest signal.
2. **dbt Cloud config on disk or in env.** `~/.dbt/dbt_cloud.yml`, a `dbt_cloud:` block, or `DBT_CLOUD_*` / `DBT_HOST` / `DBT_TOKEN` / `DBT_PROD_ENV_ID` environment variables mean the project targets dbt Platform even if the MCP is not wired up yet; wire it up or say so.
3. **A local `profiles.yml` with a warehouse target and a `dbt`/`dbtw` CLI.** This is local dbt Core.
4. **Ask.** If still unclear, ask the user one sentence: is this dbt Cloud or local dbt Core.

Say the runtime out loud once when a stakeholder is watching.

## Tool selection

| Job | dbt Core (local) | dbt Platform (Cloud, via dbt MCP) |
|---|---|---|
| Discover models, grain, columns | Read [warehouse-map.md](warehouse-map.md), then open model files and yml for what the map does not answer | `get_all_models`, `get_node_details`, `get_all_sources` (the discovery API is the map; do not grep the repo first) |
| Trace lineage and dependencies | Read `ref()` chains in the model files | `get_lineage`, `get_related_models` |
| Run ad hoc SQL (profiling, checks) | `dbtw show --inline "<sql>"` with `ref()`/`source()` | `execute_sql` (runs on platform infrastructure) |
| Preview a model before building | `dbtw show --select <model>` | `compile` then `execute_sql`, or `show` |
| Build, test, snapshot | `dbtw build --select <model>` | `build` / `run` / `test` (dbt CLI tool group, self-hosted MCP) |
| Model health, run history | Elementary report, if installed | `get_model_health`, `get_model_performance` |
| Metric asks against a semantic layer | Not available locally unless configured | `list_metrics`, `query_metrics`, `get_dimensions` |
| Generate boilerplate yml or staging | dbt bundle codegen references | `generate_model_yaml`, `generate_staging_model`, `generate_source` |

Exact tool names follow the dbt-labs dbt MCP server. The bundle's `configuring-dbt-mcp-server` skill covers setup and the environment variables that enable each tool group; this file covers which tool to prefer, not how to install it.

## dbt Platform (Cloud): the MCP is the default

When the runtime is dbt Platform:

- **Discovery replaces grepping.** The anchor step in refine-request maps the ask's nouns to real objects through `get_all_models` and `get_node_details`, not by opening files. The discovery API already holds names, descriptions, columns, and freshness; it is faster and it is current, where a checked-in `warehouse-map.md` can be stale. Fall back to reading files only for logic the metadata does not carry (the actual SQL of a transformation).
- **Queries run through `execute_sql`.** explore-data's profiling battery, validate's checks, and quick-query's answers run through `execute_sql` instead of `dbtw show --inline`. The SQL shapes are the same; only the transport changes. The warehouse guardrail file still governs cost: `execute_sql` runs against the real warehouse, so the [warehouses/](warehouses/) file for the adapter applies in full.
- **Builds run through the CLI tool group** (`build`, `run`, `test`, `compile`) when it is enabled. Selections still name models explicitly, never `+`.
- **Semantic layer first for metric asks.** If the project has a semantic layer and the ask is a defined metric, `query_metrics` answers it without hand-writing SQL. Check `list_metrics` before writing a rollup that already exists as a metric.

## dbt Core (local): CLI plus filesystem

When the runtime is local dbt Core, the existing behavior holds: discover through `warehouse-map.md` and the model files, query through `dbtw show --inline` / `--select`, build through `dbtw build`. The `dbtw` wrapper and its lock handling are a DuckDB detail covered in [warehouses/duckdb.md](warehouses/duckdb.md); a local project on another warehouse runs plain `dbt` unless it ships its own wrapper.

## Safety

- `execute_sql`, `run`, and `build` on dbt Platform touch real infrastructure and cost real money. The warehouse guardrail file is not optional in Cloud mode; it is the only thing standing between a profiling query and a large bill.
- Admin and write tools (`trigger_job_run`, `cancel_job_run`, `retry_job_run`, `clone`) change shared state. Treat them like any outward action: state what will happen and get an explicit go-ahead before firing. Never trigger a job to answer a read-only question.
- `text_to_sql` and `execute_sql` will happily run generated SQL against production. The suite's rules still apply: frame first, read the guardrail file, and run aggregates, not row dumps.
