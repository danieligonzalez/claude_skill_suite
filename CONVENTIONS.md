# Analytics engineering conventions

Conventions for a dbt analytics-engineering project, portable across warehouses (DuckDB, Snowflake, BigQuery, Redshift, Databricks) and across runtimes (local dbt Core or dbt Platform, the Cloud offering). These rules bind all SQL, models, and docs in the project.

## Warehouse and runtime

Two facts about the environment decide how every rule below runs, so establish both at the start of a session, before any SQL:

- **Which warehouse.** It sets the cost model, the SQL dialect, and what counts as an efficient query: BigQuery bills bytes scanned, Snowflake bills warehouse seconds, Redshift rewards dist and sort keys, DuckDB is local and nearly free. Identify the warehouse, then read its file in `skills/shared/warehouses/` and follow its guardrails before writing or running warehouse-touching SQL. The identification protocol is that folder's README.
- **Which runtime.** Local dbt Core runs dbt through the CLI and discovers models from the filesystem; dbt Platform (Cloud) runs dbt through the dbt MCP server and discovers models from the metadata API. On dbt Platform the MCP is the default: do not grep the whole repo for something the discovery API already knows. The protocol is in `skills/shared/runtime.md`.

Both get said out loud once when a stakeholder is watching, alongside the lane routing: the environment was identified, not assumed.

## SQL style guide

All SQL in this project follows these rules:

- Leading commas.
- No subqueries. Use CTEs and name them after what they hold.
- Tables and CTEs are referenced by their full name in join conditions and column qualifiers, never through single letter or abbreviated aliases. Write `left join dim_date on payments.payment_date = dim_date.calendar_date`, not `left join dim_date as d on p.payment_date = d.calendar_date`. A `ref()` relation already qualifies by its model name with no alias needed. Alias only when a genuine collision forces it (a self join), and then the alias is a descriptive word, not a letter.
- `/* */` for multi line comments. `--` only for short single line notes.
- The shapes and examples here are written in DuckDB idiom, the suite's reference dialect. On another warehouse, translate through the **Dialect and idiom notes** in that warehouse's file: `qualify` support, cast syntax, `date_diff` argument order, and null-safe comparison all vary.

## DRY

All code stays DRY. Every piece of logic lives in exactly one place:

- Cleaning rules run once in staging (lowercased IDs, typed dates, surrogate keys). Downstream models trust staging output and never re-apply them.
- SQL repeated across models becomes a jinja macro in `dbt/macros/`. The folder and its `_macros__macros.yml` scaffold are in place; each macro lands with its yml entry. Two models sharing the same expression is the signal to extract.
- Repeated literals (dates, thresholds, column lists) become jinja `set` variables at the top of the model, or vars in `dbt_project.yml` when shared across models.
- Check dbt_utils before writing a helper. It is installed and covers surrogate keys, date spines, and most generic tests.
- The same rule applies to docs: state a fact in one file and point to it from the others.

## Model conventions

- Folder flow is staging, then intermediate, then marts. Naming: `stg_`, `int_`, `fct_` and `dim_`. Shared helpers like `dim_date` live in utilities.
- Every `fct_` and `dim_` model gets a Kimball type when it is framed, and the type travels with the model through build and review. Facts: transaction (one row per event), periodic snapshot (one row per entity per period), accumulating snapshot (one row per process instance, updated as milestones land). Dimensions: plain, or SCD Type 2 when history must survive. Bridges resolve many to many. The type states the grain and the update behavior in one word, so say it out loud in the framing and check it in review.
- Column naming: timestamps end in `_at`, dates end in `_date`, booleans start with `is_`, `was_`, or `has_`.
- `int_` models are named for the transformation they perform (`int_payments_deduplicated`), so the name itself states why the model earns its layer. A name that could describe a mart (`int_payments`) hides the reason the model exists.
- An attribute that describes an entity (a segment, a tier, a flag) is defined once in that entity's dimension and joined from there. A mart may derive an attribute locally only when it depends on the mart's own grain or window; if a second mart could ever want it, it belongs in the dimension.
- String labels the SQL invents (case statement outputs, segment names, status rollups) read as words: initial capital, spaces not underscores. 'Has parent account', not 'has_parent_account'. Values passing through from source keep whatever staging normalized them to.
- Staging is one model per raw table: incremental merge on the natural key, filtered on `ingested_at`, surrogate key hashed from the normalized natural key.
- Materializations are a decision, not a default. Dimensions build as tables. Facts build incremental when they qualify, as tables when they do not. Views are reserved for light transformations (renames, casts, thin flags); logic with joins, windows, or aggregation materializes. These rules bind every model that lands, whatever lane the work started in: a model built mid-analysis is still a model and goes through build-model. A view in marts is always a defect; marts aggregate, and aggregation materializes.
- A fact qualifies for incremental only when all three hold: a cursor column (`ingested_at` or an event date) finds new rows, rows at the grain are never rewritten by late data or a `unique_key` merge with a stated lookback absorbs the rewrite, and a full refresh produces the same table as the incremental path. Full-grid periodic snapshots that regenerate closed periods, small dimensions, and any model whose grain rows churn do not qualify; forcing incremental onto them is a defect, table is the correct answer.
- Incremental is proven, never asserted: build twice (the second run must process nothing new on a static load and the grain counts must match), then `--full-refresh` and compare counts against the incremental result. When datasets are static (as in a local DuckDB project), an incremental model never meets a real second batch; the parity checks are then the only evidence it works and are mandatory. The command sequence lives in the build-model template. The incremental strategy itself (merge, delete+insert, or partition insert_overwrite) and any clustering or partition tuning are warehouse-specific; the model's config follows the incremental and materialization notes in `skills/shared/warehouses/`.
- Each folder has a yml docs file (`_<folder>__models.yml`). Every model and column gets a description when it lands. Tests follow the placement rule: a guarantee is tested once, at the layer that creates it, and passthrough columns are never re-tested downstream (full rules in the build-model skill). Keep descriptions concise: what the field holds and how it connects to the purpose of the table.
- Singular data tests go in `dbt/tests/`, one select per file that returns failing rows.

## Warehouse operations

Warehouse-specific operational detail lives in that warehouse's file in `skills/shared/warehouses/`, not here: the DuckDB single-writer lock, the `dbtw` wrapper, and the committed database file (duckdb.md); Snowflake warehouse sizing and the result cache (snowflake.md); BigQuery partition filters and dry-run cost estimation (bigquery.md); Redshift dist and sort keys (redshift.md); Databricks file compaction and Z-order (databricks.md). Read the file for the active warehouse. Everything else in this document is warehouse-agnostic.

## Observability

Elementary (dbt package plus the `edr` cli, optional) records run results, test results, and schemas in `main_elementary` through its own on-run-end hooks. On dbt Platform the same run and test history is available through the MCP (`get_model_health`, `get_model_performance`) with no extra package. `edr` opens its own connection to the warehouse, so on a local DuckDB project it counts as a writer: run it after dbt finishes and with the UI detached (`edr monitor` is unproven on DuckDB; use `edr report`).

## Skills

Two layers of skills operate in this repo:

- The dbt agent skills bundle from dbt-labs (`dbt@dbt-agent-marketplace`, enabled in `.claude/settings.json`) covers dbt mechanics: unit test yml spec, debugging references, command syntax.
- Repo skills in `skills/` encode the working method, built one PR at a time. The chain in runtime order: refine-request, explore-data, build-model, validate, review, document. Each skill states when to hand off to its neighbor, so scopes stay exclusive: refine decides, explore looks, build lands, validate proves, review judges, document explains.
- A seventh skill, quick-query, sits beside the chain for one-off questions. The routing test: an answer consumed once, now, by the person asking stays a query in that lane; anything reused, refreshed, or trusted by others later goes through the chain. Keepers live in `dbt/analyses/`, and the second ask of the same question becomes a model.

The full map of both layers, including which dbt bundle skills the chain uses and where their files live on disk, is in `skills/README.md`. When both layers apply, the repo skill is the entry point and the dbt bundle serves as reference material from inside it. Three rules override anything any skill says:

- How dbt runs follows the runtime (`skills/shared/runtime.md`). On local dbt Core it runs through the CLI: `dbtw` in a DuckDB-UI project (it hands off the lock), plain `dbt` otherwise. On dbt Platform (Cloud) the dbt MCP server is the default for discovery, querying, and builds. Identify the runtime before the first dbt action.
- dbt selections name models explicitly: `--select model_a model_b`. Never graph operators (`+`), never path or fqn selectors. An explicit list is auditable at a glance and cannot surprise-build half the DAG.
- Ad hoc SQL happens only inside explore-data, validate, and quick-query, always after the active warehouse's guardrail file has been read, and runs through the runtime's query path: `dbtw show --inline` with `ref()` or `source()` on local dbt Core, `execute_sql` on dbt Platform. Quick-query answers run from files in `dbt/analyses/` by name (the skill's workbench rule); the inline path covers profiling probes, validation checks, and throwaway shape checks. A question worth answering more than once is a model, not a query.

Delegation follows a manager and worker split: the main session frames, judges, and signs off; low level well defined execution (a profiling battery, writing files from an agreed sketch, running a check suite) goes to sonnet subagents that return compressed findings, never raw output. Database touching subagents run one at a time: on DuckDB there is one writer and the `dbtw` hand off is not reentrant, and on a shared cloud warehouse parallel writers race on state and multiply cost, so two subagents that query or build must never run in parallel.

Model routing follows the nuance of the job. The judgment skills stay with the strongest model in the main session: refine-request, review, document, and every framing, finding, and sign off, because their value is taste and each miss is expensive. The defined skills are sonnet work when they delegate: explore-data's profiling battery, validate's check suite, and build-model's typing once the sketch is agreed, because the spec is complete and a stronger model adds latency, not quality. The routing is worth one spoken sentence whenever a hand off happens ("this battery is fully specified, so a faster model runs it"): which model a step deserves is part of the design, and saying it shows the orchestration was chosen, not defaulted.

When working under time pressure, such as a live pairing session, the subagent hop is the one piece of ceremony to trade away: run the profiling and validation batteries inline in the main session instead of delegating. Delegation buys context economy, not correctness, so inlining runs the identical checks and changes speed, nothing else. The gates never yield to the clock: the framing contract and its go ahead, the preview before materializing, the second path to every headline number, the review sign off. Compress narration before touching a check, and cut no check at all.

## Working rules

- In a local DuckDB project, ingestion runs from `duckdb/` and dbt runs from `dbt/` (`dbtw` handles the directory itself). Other warehouses load data through their own native path (a warehouse copy or load command, dbt seeds, or external tables) and run plain `dbt`; the directory layout is the project's to set.
- A dataset that is not already in raw follows `skills/shared/new-dataset-intake.md`.
- Repo changes ship as a PR and squash merge to main. One PR per logical step.
- Docs and commit messages are plain declarative prose. No em-dashes.
