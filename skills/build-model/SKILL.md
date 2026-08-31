---
name: build-model
description: Land a dbt model from an agreed framing. Use after refine-request has stated the grain, pattern, and landing spot. Sketches the CTE chain backward from the output grain, applies the DRY check, writes the model with its yml docs and tests, and builds green through dbtw. Never starts without a stated grain.
---

# build-model

Turn an agreed framing into a built, tested, documented model. The contract from refine-request is the input; a green `dbtw build` is the exit. If there is no stated grain, stop and invoke refine-request first.

This skill applies whenever a model file is about to land, however the work started. Analysis that wants to materialize something has left the analysis lane: it goes back through refine-request for a framing and a go ahead, then lands here under all the same rules, materialization included. There is no such thing as an analysis build.

## Steps

1. **Read the contract and the brief.** The framing block from refine-request, plus any `exploration/` brief on the sources involved. Both exist before this skill starts.
2. **Sketch backward from the output.** The final CTE is the grain, almost always a group by at exactly it. For each output column, ask what derived column the group by needs; that is usually one window function or flag. Work back until the chain reaches `ref()` inputs. The CTE chain falls out before any SQL is written. Name CTEs after what they hold.

   Say the sketch out loud in plain language before any SQL exists, and after each model lands, say in two sentences what it does and why it is there. The user must be able to follow the build as it happens and defend every line afterward; a model the user cannot explain is not done, whatever the tests say.
3. **DRY check before writing.** What does staging already provide? Is there a dbt_utils helper for it? Is an expression about to appear twice? On dbt Platform, check what staging provides via `get_all_models` / `get_node_details` rather than reading files. The rules live in the CONVENTIONS.md DRY section; apply them here, at write time, not in review.
4. **Write the model.** CONVENTIONS.md SQL style: leading commas, CTEs over subqueries, `ref()` and `source()` only. Materialization follows the CONVENTIONS.md materialization rules: dimensions as tables, facts incremental when they pass the qualify test and tables when they do not, views only for light transformations. The choice was stated in the framing; say it again when the model lands, with the reason. An incremental fact declares `materialized='incremental'` with `unique_key` in its config block (the folder default stays table), and its proof sequence in the template is part of building green, not an optional extra. The incremental strategy itself (merge, delete+insert, partition `insert_overwrite`) and any clustering or partition tuning are warehouse-specific: follow the active warehouse's file in [../shared/warehouses/](../shared/warehouses/) rather than defaulting to one strategy everywhere. The dbt bundle's `adding-dbt-unit-test` skill covers unit testing `is_incremental()` branches when the incremental logic itself needs proof. Match the shape in [references/model-template.md](references/model-template.md): import CTEs first selecting only what is used, logic CTEs named after what they hold, final select at exactly the stated grain. Comments and yml descriptions use magnitudes and as-of dates, never bare exact counts; a count frozen into prose goes stale on the next load (same rule the data guide follows).
5. **Preview before materializing.** Local dbt Core: `dbtw show --select <model>` compiles and runs the model without building it. dbt Platform: `compile` then `execute_sql`, or `show`, over the MCP. Eyeball the grain and a couple of values against the contract either way. `show` needs every `ref()` to be materialized already; for a chain of new models, follow the new-chain sequence in the template instead of discovering the failure order live.
6. **Docs and tests land with the model.** Fill the folder's `_<folder>__models.yml` entry: model description, column descriptions, tests. Add or update the model's row in [../shared/warehouse-map.md](../shared/warehouse-map.md) in the same pass (the upkeep rule is stated there). Tests are intentional, not plentiful: in production every test is a query that delays the job and costs warehouse spend, so each one must defend a real invariant that could actually break.
   - A guarantee is tested once, at the layer that creates it. Staging proves the natural key; a downstream model never re-tests a passthrough column, and its yml description names the guarantor instead ("not_null guaranteed by staging"). Re-testing upstream guarantees at every layer is the same defect as re-applying cleaning rules downstream: it says nobody trusts the layer contract.
   - Downstream, a grain test earns its place only when the model's own logic could change the row set: a join re-proves `unique` on the grain key because it could fan out, an aggregation proves the new grain it creates. A pure window, flag, or rename layer changes no row counts and adds no grain test.
   - The floor, applied at the layer that creates each guarantee: `unique` and `not_null` on the grain key, `relationships` on keys that arrive from outside the model, `accepted_values` only on passthrough columns where the brief established a domain.
   - Never test what the code guarantees: no `accepted_values` on a column whose values are hardcoded by a case statement in the same model, no `not_null` on a coalesced column. A test that cannot fail is pure cost.
   - Never assert a data coincidence. An invariant must hold by design, not because today's load happens to satisfy it; a row count that matches another table only because every entity currently participates is a coincidence wearing a test's clothes.
   - When the logic is a date window, boundary math, or a case ladder, a unit test is required, not optional: the real data may never exercise the boundary, and the unit test is the only place day 90 provably behaves. Use the dbt bundle's `adding-dbt-unit-test` skill for the yml spec.
7. **Build green.** Local dbt Core: `dbtw build --select <model>`. dbt Platform: `build`, or `run` plus `test`, over the MCP CLI tool group. Either way, models named explicitly, never `+` (CONVENTIONS.md rule). Build, not run: it executes the tests in the same pass. A red test here is a finding, not an obstacle; read it before touching the test.

## Delegation

The sketch is the manager's job; the typing is not, but delegation has a threshold. Hand off only when the sketch is meaningfully smaller than the SQL it produces: a multi-model chain, or mechanical repetition across files. A single model whose sketch is essentially the SQL column by column gets written directly; a hand off there adds tokens and minutes and removes the manager's feel for the code without saving anything.

When the threshold is met: hand a sonnet subagent the sketch, the target file paths, and the yml entry shape; it writes the model and the yml, runs the preview and the build per the template's command sequence for the active runtime (`dbtw` locally, the MCP tools on dbt Platform), and returns the build result plus the preview rows, not the logs. The manager judges the preview against the contract and owns the DRY call and the hand off to validate. One building subagent at a time (CONVENTIONS.md delegation rule).

## Output

Stated in conversation when the build is green: the model name, its grain, what it reads from, the tests that now guard it, and anything the preview surfaced that the requester should know.

## Hand off

- No stated grain or landing spot: back to **refine-request**. This skill does not frame work.
- A source fact is missing mid-build (is this key unique, what are these statuses): **explore-data**, then resume.
- Build errors beyond the obvious: the dbt bundle's debugging reference inside `using-dbt-for-analytics-engineering`.
- Green build: hand to **validate**. Built is not the same as right; validate proves the numbers.
