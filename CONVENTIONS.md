# Analytics engineering conventions

Conventions for a local dbt + DuckDB analytics-engineering workspace. DuckDB stores the data, dbt core transforms it. These rules bind all SQL, models, and docs in the project.

## SQL style guide

All SQL in this repo follows these rules:

- Leading commas.
- No subqueries. Use CTEs and name them after what they hold.
- Tables and CTEs are referenced by their full name in join conditions and column qualifiers, never through single letter or abbreviated aliases. Write `left join dim_date on payments.payment_date = dim_date.calendar_date`, not `left join dim_date as d on p.payment_date = d.calendar_date`. A `ref()` relation already qualifies by its model name with no alias needed. Alias only when a genuine collision forces it (a self join), and then the alias is a descriptive word, not a letter.
- `/* */` for multi line comments. `--` only for short single line notes.

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
- Incremental is proven, never asserted: build twice (the second run must process nothing new on a static load and the grain counts must match), then `--full-refresh` and compare counts against the incremental result. This repo's datasets are static, so an incremental model never meets a real second batch; the parity checks are the only evidence it works and are mandatory. The command sequence lives in the build-model template.
- Each folder has a yml docs file (`_<folder>__models.yml`). Every model and column gets a description when it lands. Tests follow the placement rule: a guarantee is tested once, at the layer that creates it, and passthrough columns are never re-tested downstream (full rules in the build-model skill). Keep descriptions concise: what the field holds and how it connects to the purpose of the table.
- Singular data tests go in `dbt/tests/`, one select per file that returns failing rows.

## The database file and its lock

- DuckDB allows one writer on the file at a time. `profiles.yml` sets `keep_open: false` plus connect retries, so terminal dbt, the VS Code dbt Power User extension, and `ingest.py` hand the lock around on their own. Do not reach for killing lock holders as a first resort; the README has the two recovery moves for when something is genuinely stuck.
- The DuckDB UI is the exception: it holds the lock while `warehouse` is attached. Run dbt through `dbt/dbtw` (aliased to `dbtw` in the shell), which detaches the database from a running UI over the UI server's local HTTP endpoint, runs dbt, and re-attaches. With no UI running it is a plain dbt call, so `dbtw` is always the right way to run dbt.
- `dbtw` does not cover `ingest.py`, Power User query previews, or `edr`. For those, run the detach cell in the UI first: `USE memory; DETACH warehouse;`.
- The UI must start bare (`duckdb -ui`), never with the database file as an argument. A main database can never be detached.
- `warehouse.duckdb` is committed to the repo and churns whenever anything writes to it. Let the churn ride along with the next real commit; never commit it alone.

## Observability

Elementary (dbt package plus the `edr` cli) records run results, test results, and schemas in `main_elementary` through its own on-run-end hooks. The README has the report command. `edr` opens its own connection, so it counts as a writer: run it after dbt finishes and with the UI detached. `edr monitor` is unproven on DuckDB; use `edr report` only.

## Skills

Two layers of skills operate in this repo:

- The dbt agent skills bundle from dbt-labs (`dbt@dbt-agent-marketplace`, enabled in `.claude/settings.json`) covers dbt mechanics: unit test yml spec, debugging references, command syntax.
- Repo skills in `skills/` encode the working method, built one PR at a time. The chain in runtime order: refine-request, explore-data, build-model, validate, review, document. Each skill states when to hand off to its neighbor, so scopes stay exclusive: refine decides, explore looks, build lands, validate proves, review judges, document explains.
- A seventh skill, quick-query, sits beside the chain for one-off questions. The routing test: an answer consumed once, now, by the person asking stays a query in that lane; anything reused, refreshed, or trusted by others later goes through the chain. Keepers live in `dbt/analyses/`, and the second ask of the same question becomes a model.

The full map of both layers, including which dbt bundle skills the chain uses and where their files live on disk, is in `skills/README.md`. When both layers apply, the repo skill is the entry point and the dbt bundle serves as reference material from inside it. Three rules override anything any skill says:

- dbt always runs through `dbtw`, never plain `dbt` and never MCP tools. The lock section above explains why.
- dbt selections name models explicitly: `--select model_a model_b`. Never graph operators (`+`), never path or fqn selectors. An explicit list is auditable at a glance and cannot surprise-build half the DAG.
- Ad hoc SQL happens only inside explore-data, validate, and quick-query, and runs through `dbtw show` with `ref()` or `source()`. Quick-query answers run from files in `dbt/analyses/` by name (the skill's workbench rule); `--inline` covers profiling probes, validation checks, and throwaway shape checks. A question worth answering more than once is a model, not a query.

Delegation follows a manager and worker split: the main session frames, judges, and signs off; low level well defined execution (a profiling battery, writing files from an agreed sketch, running a check suite) goes to sonnet subagents that return compressed findings, never raw output. Database touching subagents run one at a time: DuckDB has one writer and the dbtw hand off is not reentrant, so two subagents that query or build must never run in parallel.

Model routing follows the nuance of the job. The judgment skills stay with the strongest model in the main session: refine-request, review, document, and every framing, finding, and sign off, because their value is taste and each miss is expensive. The defined skills are sonnet work when they delegate: explore-data's profiling battery, validate's check suite, and build-model's typing once the sketch is agreed, because the spec is complete and a stronger model adds latency, not quality. The routing is worth one spoken sentence whenever a hand off happens ("this battery is fully specified, so a faster model runs it"): which model a step deserves is part of the design, and saying it shows the orchestration was chosen, not defaulted.

When working under time pressure, such as a live pairing session, the subagent hop is the one piece of ceremony to trade away: run the profiling and validation batteries inline in the main session instead of delegating. Delegation buys context economy, not correctness, so inlining runs the identical checks and changes speed, nothing else. The gates never yield to the clock: the framing contract and its go ahead, the preview before materializing, the second path to every headline number, the review sign off. Compress narration before touching a check, and cut no check at all.

## Working rules

- Ingestion runs from `duckdb/`. dbt runs from `dbt/` (`dbtw` handles the directory itself).
- A dataset that is not already in raw follows `skills/shared/new-dataset-intake.md`: read_csv_auto to query it where it sits, the ingest.py path the moment it feeds a model.
- Repo changes ship as a PR and squash merge to main. One PR per logical step.
- Docs and commit messages are plain declarative prose. No em-dashes.
