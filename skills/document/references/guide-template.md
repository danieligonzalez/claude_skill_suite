# Data guide template

One file per mart or pipeline in `docs/`, this shape every time. The example continues fct_payments_monthly; values are illustrative.

```markdown
# fct_payments_monthly

**What it answers.** How much we billed and how much we collected, by calendar month, including months with no activity.

**The pipeline.** raw.payments loads as ingested (all varchar, plus source and ingested_at). stg_payments types the columns, lowercases IDs and status, and merges incrementally on payment_id. fct_payments_monthly aggregates staging to the month and left joins onto the dim_date month spine so empty months survive as zeros.

**Grain and guarantees.** One row per calendar month. month_start_date is unique and never null (tested). Amounts are never null; empty months carry zeros (spine join plus coalesce). Every payment in staging lands in exactly one month.

**How problems are caught.** Every update runs automated checks before the numbers land anywhere. The checks confirm the table stays exactly one row per month with nothing duplicated or dropped, that every payment traces back to a real customer, and that the calculation logic still produces the right answers on hand-built examples with known answers. If any check fails, the update stops and the previous numbers stay in place until an engineer resolves it, so a data problem shows up as a delay, not as a wrong number in a meeting. Every run and every check result is also recorded, so a quiet change in the source data surfaces as an alert instead of a silent shift in the trend.

**Expected results.** ~32 rows as of the 2026-08 load, one per month from 2024-01 through 2026-08, including 3 zero months. Known exclusion: 41 orphaned payments are quarantined in staging and not counted here; see exploration/payments.md.

**How to use it.**

    select month_start_date
         , collected_amount / nullif(billed_amount, 0) as collection_rate
    from main_marts.fct_payments_monthly
    order by month_start_date
```

Rules for filling it:

- Written for the stakeholder who will act on the numbers, not the engineer who built the model. Plain declarative sentences; jargon gets a plain gloss the first time it appears or gets cut.
- Two minute read. If a section runs long, the model is too complicated or the guide is doing the yml's job; column detail stays in the folder yml, link there instead.
- The pipeline section is one line per model, each line saying what that step does to the data and nothing else.
- Every guarantee names its backing: (tested), or the mechanism (spine join plus coalesce). An unbacked guarantee routes to build-model for the test before the guide ships.
- The problems section names what the checks protect, never test names or tools: "one row per month with nothing duplicated", not "unique test on month_start_date". It always ends with what happens on failure. The bar: a VP with no SQL could repeat it to answer "how do we know these numbers are right".
- Expected results use magnitudes and as-of dates, never bare exact counts; known exclusions link to the exploration brief that discovered them. Partial first or last periods get named as partial, or the edge months read as real dips.
- When the mart aggregates a row level flag, the how-to-use section names the model and flag column holding the actionable rows, so the reader can pull the list behind the totals.
- The example select answers a question someone would actually ask, at the model's grain, in repo SQL style.
