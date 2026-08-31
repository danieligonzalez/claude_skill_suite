# Query shapes

Shared by explore-data (profiling, before building) and validate (checks, after building). Every query runs as `dbtw show --inline "<sql>"`: double quotes around the SQL for the shell, single quotes inside for SQL strings, jinja braces need no escaping. That is the local dbt Core path; on dbt Platform the same SQL runs through `execute_sql` instead. See `runtime.md` for which path applies. Aggregates return one row, so the default show limit never truncates them. Raw columns are all varchar; cast before date or numeric work. Examples below use the real sources and an example mart, `fct_payments_monthly`; substitute the model at hand.

These shapes are written in DuckDB idiom, the reference dialect for this doc. On another warehouse, translate through that warehouse's dialect and idiom notes and reach for its cheap probes section (metadata for row counts, partition-scoped windows) instead of scanning the table. See `warehouses/README.md`.

## Grain proof

The first query on any table, source or model. Equal counts prove the grain; unequal counts are the finding.

```sql
select count(*) as row_count
     , count(distinct customer_id) as key_count
from {{ source('raw', 'customers') }}
```

## Profiling shapes (explore-data)

Duplicate keys, and whether copies agree (versions = 1 means identical copies, more means conflicting data):

```sql
select customer_id
     , count(*) as copies
     , count(distinct first_name || '|' || last_name || '|' || created_at) as versions
from {{ source('raw', 'customers') }}
group by 1
having count(*) > 1
order by copies desc
limit 10
```

Duplicate spacing in time, when duplicates exist and carry a date. A span of zero means the copies land together; a spread is a mechanism clue (a retry lands same-day, a replayed batch lands days later) and goes in the brief:

```sql
with duplicate_groups as (

    select customer_id
         , service_date
         , datediff('day', min(cast(payment_date as date)), max(cast(payment_date as date))) as day_span
    from {{ source('raw', 'payments') }}
    group by 1, 2
    having count(*) > 1

)

select day_span
     , count(*) as group_count
from duplicate_groups
group by 1
order by 1
```

Orphans, both directions:

```sql
select count(*) as payments_without_customer
from {{ source('raw', 'payments') }} as pay
where not exists (
    select 1
    from {{ source('raw', 'customers') }} as cust
    where lower(cust.customer_id) = lower(pay.customer_id)
)
```

Null profile in one pass (count skips nulls; blank strings are a separate check in raw varchar data):

```sql
select count(*) as row_count
     , count(*) - count(customer_id) as customer_id_null
     , sum(case when trim(customer_id) = '' then 1 else 0 end) as customer_id_blank
     , count(*) - count(amount) as amount_null
from {{ source('raw', 'payments') }}
```

Value domain (low cardinality columns only):

```sql
select status
     , count(*) as row_count
from {{ source('raw', 'payments') }}
group by 1
order by 2 desc
```

Date coverage:

```sql
select min(cast(payment_date as date)) as first_date
     , max(cast(payment_date as date)) as last_date
     , count(distinct cast(payment_date as date)) as days_present
from {{ source('raw', 'payments') }}
```

Days present against the span between first and last flags gaps without listing them. List actual gap dates only if the framing needs them.

## Check shapes (validate)

The counts walk, one query so the steps sit side by side. Add a step per dedup or filter the pipeline applies; every adjacent difference gets a stated reason:

```sql
select 'raw' as step, count(*) as row_count
from {{ source('raw', 'payments') }}
union all
select 'staging', count(*)
from {{ ref('stg_payments') }}
union all
select 'mart', sum(payment_count)
from {{ ref('fct_payments_monthly') }}
```

Boundary rows: pull the first and last rows at the model's grain and compare against the source range from the date coverage shape above. Then the edge groups:

```sql
select count(*) as zero_payment_months
from {{ ref('fct_payments_monthly') }}
where payment_count = 0
```

```sql
select count(*) as null_date_payments
from {{ ref('stg_payments') }}
where payment_date is null
```

If null date rows exist, say where the mart puts them. Silently dropped is the wrong answer unless the contract says so.

Second path to the total: compute the headline number without the model and compare. Exact equality; close enough is a bug with a hiding place.

```sql
select sum(case when status = 'paid' then amount else 0 end) as collected_direct
from {{ ref('stg_payments') }}
```

```sql
select sum(collected_amount) as collected_mart
from {{ ref('fct_payments_monthly') }}
```

Idempotency (incremental models): `dbtw build --select <model>` twice in a row; the grain proof must return the same counts both times.
