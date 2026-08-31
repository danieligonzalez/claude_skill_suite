# New dataset intake

What to do when a file that is not customers or payments lands, when working under time pressure or otherwise. Two paths, chosen by whether the data is about to be load-bearing.

This file covers the **local DuckDB** path. Identify the warehouse and runtime first (see `warehouse-map.md` and `runtime.md`); the steps below assume both are local.

## Local DuckDB path

### Fast path: query it where it sits (under a minute)

DuckDB reads a CSV without loading it, so the file is queryable through the normal mechanism inside quick-query or explore-data:

```
dbtw show --inline "select count(*) from read_csv_auto('/absolute/path/to/file.csv')"
```

- Absolute path only. The wrapper runs dbt from `dbt/`, so a relative path resolves somewhere surprising.
- `read_csv_auto` infers types. When inference looks wrong, re-read with `all_varchar=true` and cast by hand, the same trust level as the raw layer.
- Say the trade out loud when a requester is watching: queryable now, but nothing downstream can `ref()` it and nothing tests it. This path answers questions; it never feeds models.

### Durable path: land it in raw (a few minutes)

The moment the file feeds a model, it stops being a fast-path file:

1. Copy the CSV to `duckdb/data/<table>.csv`. The filename becomes the raw table name.
2. Add the name to the `TABLES` list in `duckdb/ingest.py`. The loader is generic: all varchar, `source` and `ingested_at` added at load. The `source` value is hardcoded in the script; change it when the file comes from a different system than crm.
3. Run the ingest with the UI detached (`USE memory; DETACH warehouse;` in the detach cell first; ingest.py is not covered by dbtw): `python ingest.py` from `duckdb/`. Re-attach after.
4. Add the table to `_raw__sources.yml` with column descriptions.
5. One `stg_` model per raw table, per the staging conventions in CONVENTIONS.md.
6. explore-data profiles it before anything builds on it; the brief lands in the repo root `exploration/`.

The routing sentence for a live working session: "I can profile this where it sits right now; the moment we model on it, I land it in raw so staging cleans it once."

## Other warehouses and dbt Platform

On a cloud warehouse, there is no local file system to read a CSV from, so intake goes through the warehouse's own load path: a native COPY/load command, an external or staged table over object storage, or a dbt seed for a small static file. From there the steps are the same shape as above: define a source, then add one `stg_` model per source table. Read the adapter's file in `warehouses/README.md` for the exact load mechanism and its cost.

On dbt Platform, sources are defined in the project as usual, but discovery runs through the dbt MCP (`get_all_sources`) rather than by opening yml files, per `runtime.md`.
