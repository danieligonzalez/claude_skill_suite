# Skills index

The map of both skill layers. The routing rules live in the CONVENTIONS.md Skills section; this file is where to see what exists and what each piece is for.

## Shared references

Two files under `shared/` set the environment every other skill assumes, and get read before any warehouse-touching SQL:

- `shared/warehouses/`: one file per adapter (DuckDB, Snowflake, BigQuery, Redshift, Databricks) with its cost model, efficiency guardrails, cheap probes, and dialect notes. Identify the warehouse, then read its file.
- `shared/runtime.md`: dbt Core (local, CLI plus filesystem) vs dbt Platform (Cloud, the dbt MCP server is the default for discovery and querying). Identify the runtime the same way, before choosing which tools to reach for.

## The repo chain

| Skill | Job | Hands off to |
|---|---|---|
| refine-request | Frame the ask: grain, clarifying questions, pattern, landing decision | explore-data, build-model |
| explore-data | Profile sources, write the brief to `exploration/` | refine-request, build-model |
| build-model | Land the model with docs and tests, green `dbtw build` | validate |
| validate | Counts walk, boundaries, second path, traps scan | build-model, review, document |
| review | Ranked findings on design and sustainability | build-model, validate, document |
| document | yml descriptions plus a data guide in `docs/` | review, build-model |

## The one-off lane

| Skill | Job | Hands off to |
|---|---|---|
| quick-query | Answer a one-off question: one-line frame, repo-style SQL, self-check, answer with assumption | explore-data, validate, refine-request |

Beside the chain, not in it. Routing: consumed once by the asker stays here; anything reused goes through the chain, and the second ask of the same question becomes a model.

## The dbt bundle

`dbt@dbt-agent-marketplace` version 1.4.1, installed project scoped to this repo. The skill files live at `~/.claude/plugins/cache/dbt-agent-marketplace/dbt/1.4.1/skills/`; open any skill's `SKILL.md` there to read it, or use the `/plugin` panel in an interactive session. Update the pin through `/plugin`.

| Skill | Job | Used here |
|---|---|---|
| using-dbt-for-analytics-engineering | Core dbt workflow plus reference guides: planning models, discovering data, writing data tests, debugging errors, impact evaluation, writing documentation | Yes. build-model and document lean on its references. It may also self trigger on dbt work; that is fine, it complements the chain |
| adding-dbt-unit-test | Unit test yml spec, examples, incremental special cases | Yes. build-model delegates unit test mechanics to it |
| running-dbt-commands | Command syntax, selectors, flags | Selector syntax only. On dbt Core its preference for plain dbt is overridden by the dbtw rule; on dbt Platform builds go through the MCP CLI tool group instead per `shared/runtime.md` |
| fetching-dbt-docs | Look up dbt documentation | Ad hoc, when a dbt question comes up |
| using-dbt-state | State based selection (state:modified) | Rarely. Single dev repo, full builds are cheap here |
| building-dbt-semantic-layer | MetricFlow semantic models | Not used. No semantic layer in this repo |
| answering-natural-language-questions-with-dbt | Query the semantic layer | Not used. Same reason |
| working-with-dbt-mesh | Model versions, cross project refs | Not used. Single project, no external consumers |
| configuring-dbt-mcp-server | Set up the dbt MCP server | Yes, on dbt Platform. The MCP is the default runtime path in Cloud mode: `get_all_models` / `get_node_details` / `get_all_sources` for discovery, `execute_sql` for queries, `build` / `run` / `test` for builds. Local dbt Core still runs on the CLI (`dbtw`/`dbt`); see `shared/runtime.md` for the split |
| troubleshooting-dbt-job-errors | dbt platform job failures | Used when running on dbt Platform; not applicable to a local-only project |
