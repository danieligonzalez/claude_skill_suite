# Model template and worked sketch

Continues the worked example from refine-request: `fct_payments_monthly`, one row per calendar month.

## The backward sketch, written before the SQL

- Final CTE: a group by at the month grain producing billed_amount and collected_amount. What does it need? Every payment tagged with its month and a collected flag.
- The tag needs: payment_date truncated to month (staging already cast it; do not re-cast), and collected = status paid (staging already lowercased status; trust it).
- Months with zero payments must appear, so the final select starts from the `dim_date` month spine and left joins the aggregate onto it.

Chain, named after what each holds: `months`, `payments`, `payments_by_month`, final select. Four CTEs, no windows, done before a line of SQL exists.

## The template

Import CTEs first, one per ref, selecting only what the model uses. Logic CTEs next. Final select last, at exactly the stated grain. Column names from other models are illustrative; check the folder yml before trusting them.

```sql
/* fct_payments_monthly: one row per calendar month across the dim_date range.
   Billed sums all payments in the month; collected sums status = paid.
   Months with no payments appear with zeros (dim_date spine). */

with months as (

    select distinct month_start_date
    from {{ ref('dim_date') }}

)

, payments as (

    select payment_id
         , date_trunc('month', payment_date) as month_start_date
         , amount
         , status
    from {{ ref('stg_payments') }}

)

, payments_by_month as (

    select month_start_date
         , sum(amount) as billed_amount
         , sum(case when status = 'paid' then amount else 0 end) as collected_amount
         , count(*) as payment_count
    from payments
    group by 1

)

select months.month_start_date
     , coalesce(payments_by_month.billed_amount, 0) as billed_amount
     , coalesce(payments_by_month.collected_amount, 0) as collected_amount
     , coalesce(payments_by_month.payment_count, 0) as payment_count
from months
left join payments_by_month
    on months.month_start_date = payments_by_month.month_start_date
```

## The yml entry, same pass as the SQL

Into `_marts__models.yml`, following the scaffold's commented example:

```yaml
  - name: fct_payments_monthly
    description: One row per calendar month across the dim_date range. Billed sums all payments in the month, collected sums paid payments, months without payments appear with zeros.
    columns:
      - name: month_start_date
        description: First day of the month. Grain key.
        tests:
          - not_null
          - unique
      - name: collected_amount
        description: Sum of payments with status paid in the month. Zero for months without payments.
        tests:
          - not_null
```

## The command sequence

Models are always named explicitly, never selected with `+` (CONVENTIONS.md rule).

One model whose refs all exist:

```
dbtw show --select fct_payments_monthly      # preview against the contract before materializing
dbtw build --select fct_payments_monthly     # build plus tests in one pass
```

Preview first, build second, and build (never run) so the tests fire in the same pass. If the preview grain or a spot value disagrees with the contract, fix before building; a materialized wrong model is churn in the committed database file.

A chain of new models (say a new `int_x` feeding a new `fct_y`) breaks that sequence two ways: `show` fails because its refs are not materialized yet, and a plain `build` fails because dbt's eager indirect selection pulls in tests on downstream models that do not exist (Catalog Error). The sequence for a new chain:

```
dbtw build --select int_x --indirect-selection empty   # dependency order, downstream tests deferred
dbtw show --select fct_y --indirect-selection empty    # refs exist now; preview against the contract
dbtw build --select int_x fct_y                        # full pass, every test fires
```

Build upstream first with `--indirect-selection empty`, preview the downstream model once its refs exist, then one explicit full-selection build so all tests run together.

## The incremental proof sequence

Runs whenever a model materializes as incremental (CONVENTIONS.md materialization rules). The datasets here are static, so an incremental model never meets a real second batch; these three builds are the only evidence the incremental path works.

```
dbtw build --select fct_x                  # first build, loads everything
dbtw build --select fct_x                  # second build must process nothing new; grain counts identical
dbtw build --select fct_x --full-refresh   # rebuild from scratch; counts must match the incremental result
```

Run the grain proof (query-shapes, first shape) after the second and third builds and compare. A second build that grows the table means the merge is not idempotent; a full refresh that disagrees with the incremental result means the `is_incremental()` branch computes something different from the full path. Either one is a defect in the model, not in the check. `show` needs the flag too: eager indirect selection pulls unbuilt test nodes into the selection and fails the preview even though `show` runs no tests.
