# Warehouse map

*Example map. Replace the rows below with your project's own objects; the shape is the point.*

State the project's warehouse and runtime at the top, for example "Snowflake, dbt Platform". On dbt Platform, the discovery API (via the dbt MCP) is the live map and this file is a secondary convenience; on local dbt Core, this file is the primary one-page map.

The one-page answer to "what do we already have". refine-request reads this during the anchor step to map the ask's nouns to real tables without a live excavation; explore-data and build-model consult it before touching model files. It is a map, not documentation: one row per object, grain, the columns that matter, the guarantees that hold. Column-level detail stays in the folder yml files (DRY rule); this file only says enough to route a request.

Upkeep rule: whenever a model lands or changes shape, the same change updates its row here. A stale map misroutes every request that follows, so the map update rides in the same commit as the model. build-model step 6 and the document skill both point at this rule; this is the one place it is stated.

## Sources (schema `raw`, loaded by ingest.py, all columns varchar, `ingested_at` added at load)

| object | grain | columns |
| --- | --- | --- |
| raw.customers | one row per customer | customer_id, first_name, last_name, email, parent_customer_id (nullable, self-reference to customer_id), ingested_at |
| raw.payments | one row per payment posting | payment_id, customer_id (UPPERCASE in source), service_date, payment_date, amount, ingested_at |

## Staging (schema `main_staging`, incremental merge on the natural key, typed and normalized)

| model | grain | key facts |
| --- | --- | --- |
| stg_customers | one row per customer | customer_sk (hashed), customer_id lowercased unique not_null (tested), parent_customer_id lowercased. Names, email passthrough. |
| stg_payments | one row per payment posting | payment_sk (hashed), payment_id lowercased unique not_null (tested), customer_id lowercased, service_date and payment_date cast to date, amount cast to decimal(10,2). Five figures of rows. |

## Utilities (schema `main_marts`)

| model | grain | key facts |
| --- | --- | --- |
| dim_date | one row per calendar_date | Spine spanning the payment window. month_start_date, day_name, is_weekend, is_holiday + holiday_name (11 US federal holidays, actual observed dates, 2024 calendar pinned by a singular test). Join it instead of re-deriving calendar logic. |

## Intermediate (schema `main_staging`)

None yet. Models here are named for the transformation they perform (int_payments_deduplicated, not int_payments).

## Marts (schema `main_marts`)

None yet.

## Known data facts worth remembering before profiling

- Raw payments customer_id is uppercase; staging lowercases both sides so the tables join. Anti-joins on raw without normalizing case show 100% orphans and mean nothing.
- exploration/ briefs at the repo root record earlier profiling; check for a brief before re-running one.
