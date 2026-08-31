# Findings template

The review output, same shape every time. Findings below are illustrative examples of each tier, written against a hypothetical mart.

```
Review of fct_payments_monthly and its path from staging.
Checked: conventions, DRY, grain and joins, coverage, sustainability, placement.

Blocking:
- Grain and joins, fct_payments_monthly line 34: payments join customers on a column
  whose uniqueness is never tested; one duplicate customer silently doubles that
  customer's amounts. Fix: unique test on the join key, or dedup upstream in staging.

Should fix:
- DRY, three models: the literal 'paid' appears in fct_payments_monthly,
  fct_collections, and int_payment_flags. A new status casing breaks them
  independently. Fix: a paid_statuses var in dbt_project.yml.

Notes:
- Sustainability, fct_payments_monthly: payment_count has no downstream consumer
  yet. Keep, it is one aggregate and the counts walk uses it.

Clean areas: conventions, coverage, placement.

Validation queries to run yourself (DuckDB UI):

-- grain: should return equal counts
select count(*) as row_count, count(distinct month_start_date) as key_count
from main_marts.fct_payments_monthly;

-- headline: should match the collected total stated in validate
select sum(collected_amount) from main_marts.fct_payments_monthly;

-- the blocking finding, live: returns zero rows once the join key is clean
select customer_id from main_staging.stg_customers
group by 1 having count(*) > 1;

Sign off when you are satisfied and I will write the guide.
```

Rules:

- Every finding carries all four parts: area, location (model and line for code), why it matters stated as the failure it enables, and a one line fix.
- The why is a consequence, not a rule citation. "Breaks convention 3" is not a why; "one duplicate customer silently doubles amounts" is.
- Rank by consequence: wrong numbers possible = blocking, costs real time later = should fix, worth knowing = note.
- Name the clean areas at the end. A review that only lists problems hides how much was checked.
- A fully clean review still states the checked list and the clean areas, then says so in one line.
- The validation queries close every review, clean or not: two to four, plain schema-qualified SQL the AE pastes into the DuckDB UI, each with its expected result stated up front. Then ask for sign off and stop; document waits for it.
