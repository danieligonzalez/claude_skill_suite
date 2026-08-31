# Worked example

The ask, as a stakeholder would say it: "We want to see, for each month, how much we billed and how much we actually collected."

**Step 1, anchor.** The requester is finance; the answer feeds the monthly close review, and a weak collection month triggers a follow-up push. Every noun in the ask has a home: "billed" and "collected" both map to `payments.amount` plus `status`, "month" maps to a date on the payment. If a noun had no home (an ask about services in a warehouse with no services table), that mismatch would be the first question, never a silent substitution.

**Step 2, route the lane.** Said to the requester in one sentence: "The close review reads this every month, so it lands as a model, not a one-off query." If the ask had been a one-time look, the work would route to quick-query and stop here.

**Step 3, grain.** Said out loud before any SQL thinking: "One row per calendar month, with billed_amount and collected_amount." That sentence is the correctness spec and the query shape.

**Step 4, the questions worth asking.** Two, because they change the design, stated as decision blocks:

> 1. Which date drives the month?
>    a) The paid date, when the money moved (my default: "collected" reads as cash timing)
>    b) The created date, when the charge was raised
>    Why it matters: changes the join and the meaning of every number in the table.
>
> 2. Do refunds subtract?
>    a) Refunds net against their month (my default: a close review wants net)
>    b) Gross amounts, refunds reported separately
>    Why it matters: turns a filter into conditional aggregation with a sign.

Not asked: "how many rows are there", "are there duplicate payments". The data answers those, and that is explore-data's job.

**Step 5, pattern.** Conditional aggregation at the month grain, the aggregate plus filter family. No windows needed. If the ask had been "top customers per month", it becomes top N per group and the idiom changes with it.

**Step 6, landing.** Walk the landing rules below. The reusable asset is the payment grain fact, not the monthly rollup: `fct_payments` (one row per payment, typed amounts, clean status) serves every future payment question. The monthly rollup lands as `fct_payments_monthly` reading `fct_payments` and the `dim_date` spine, so months with zero payments appear as zeros instead of vanishing.

**The framing block as stated to the requester:**

- Deliverable: monthly billed and collected amounts across the full data range, feeding the monthly close review.
- Grain: one row per calendar month.
- Type: `fct_payments` is a transaction fact; `fct_payments_monthly` is a periodic snapshot at the month grain.
- Pattern: conditional aggregation at the month grain.
- Lands as: `fct_payments` (payment grain) in marts, plus `fct_payments_monthly` reading it and `dim_date`.
- Materialization: `fct_payments` incremental on `ingested_at` merging on payment_id (qualifies: cursor exists, a payment row is corrected in place by the merge, full refresh parity to be proven). `fct_payments_monthly` as a table: it regenerates the full month grid off the spine, so closed months rewrite and incremental does not qualify.
- Open questions: the two decision blocks above, awaiting answers.

**Step 7, the go ahead.** "That is the plan; anything you would change before I build?" The answers to the decision blocks and the plan approval arrive together, and only then does build-model start.

# Landing rules

Walk top to bottom, first match wins:

1. A mart already answers it at the same grain: extend that mart with a column. Never create a sibling model that shares a grain with an existing one.
2. A mart answers it at a different grain: new model reading the existing mart, not a rebuild from staging.
3. The logic would serve two or more marts: intermediate model, and the marts read it.
4. Nothing exists yet: new mart at the most reusable grain, usually the entity or event grain, with rollups reading it. The rollup grain someone asked for today is rarely the grain tomorrow's question needs.
5. Genuinely a one off diagnostic ("does this data have X"): not a model. That is quick-query, explore-data, or validate territory.
