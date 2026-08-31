---
name: validate
description: "Prove a built model or a number before it is reported. Use after build-model goes green, or before any number leaves the conversation. Runs the counts walk from source to final, boundary rows, a second path to the same total, and the traps checklist, then offers targeted anomaly detection tests before hand off. The output is a validation statement that makes the number defensible."
---

# validate

Built is not the same as right. This skill proves the output against the contract so the number never has to be walked back. Schema tests already passed in the build; this is the layer they cannot cover: reconciliation, boundaries, and a second opinion.

Ad hoc SQL is allowed here, through the runtime's query path: local dbt Core via `dbtw show --inline` with `ref()`, dbt Platform via `execute_sql` (CONVENTIONS.md rule; identify the runtime per [../shared/runtime.md](../shared/runtime.md)). Before running the checks, read the active warehouse's guardrail file under [../shared/warehouses/](../shared/warehouses/README.md): the counts-walk and second-path SQL follow that file's dialect notes, and its cheap-probes section covers cheaper alternatives on large tables.

## Two shapes of work arrive here

A built model from the chain, or an ad hoc analysis from quick-query. The proof logic is identical; the check list scales to what is being defended.

- **A model buildout** runs all four checks below, the idempotency check when the model is incremental, and ends with the anomaly coverage offer.
- **An ad hoc analysis** proves the answer, not a model. The counts walk runs through the query's own filters (source rows, rows surviving each filter or join, rows feeding the answer). The second path recomputes the headline number a different way and must agree exactly. Boundary rows cover the claim's edges (the first and last period in the window, a tie, a zero or singleton group). The traps scan reads the analysis SQL. Idempotency does not apply because nothing materializes, and the anomaly offer does not fire because no recurring asset lands; that conversation happens when the second ask turns the analysis into a model. The validation statement keeps the same shape, shorter.

## The checks

Run all four on a new model. On a changed model, run the ones the change touches. On an analysis, run them in the scaled form above.

1. **The counts walk.** Source rows, rows after dedup, rows after each filter, final rows. Every loss between steps gets a stated reason. An unexplained loss is a bug until explained. This walk, said out loud, is what makes a number defensible to an executive.
2. **Boundary rows.** First row, last row, a singleton group, a tie, a null in a join or filter column. Pull each and compare against the grain contract from refine-request. This is where last inch bugs die. A boundary that returns no rows is a finding of its own: report it as untested, never as passed. A check that could not fire proves nothing; name the edge the data never exercised (no rows near day 90 means the 90 day boundary is unverified) and say whether a unit test covers it instead. This distinction is the one most likely to turn a false pass into an honest gap.
3. **Second path to the total.** Compute the headline number a different way: a different route through the DAG, or a different aggregation order (sum of a grouped total against the flat total). The two paths must agree exactly, not roughly.
4. **Traps scan.** Walk [references/traps.md](references/traps.md) against the model's SQL. Each trap is a one line yes or no.

For incremental models, add the idempotency check: build twice in a row, counts must not change.

The exact SQL for each check is in [../shared/query-shapes.md](../shared/query-shapes.md), check shapes section, shared with explore-data so the two skills never drift apart on how a grain is proven.

## Delegation

The checks are fully specified, so they delegate cleanly: one sonnet subagent runs all four serially and returns each check's numbers and pass or fail, never query output. Analysis mode checks are small enough that delegation buys nothing; run them inline. The manager judges what the numbers mean: whether a stated reason for a row loss actually holds is a judgment call, not a query. Never split the checks across parallel agents (CONVENTIONS.md delegation rule: one writer, one database touching agent at a time). In a live working session, skip the hand off and run the checks inline; every check still runs (CONVENTIONS.md pacing rule).

## Output

A validation statement in conversation: each check run, what it proved, with the numbers. If everything passed, say what would still be worth watching. If a check failed, the failure is the deliverable; report it before any fix. Follow [references/statement-template.md](references/statement-template.md), which shows both a passing and a failing statement; match its shape every time.

## Anomaly coverage (the offer is mandatory, the tests are optional)

Schema tests prove structure and validation proves today's numbers; neither watches the metric itself drift on a future load. So on a buildout, after the validation statement, ask the requester one question: do they want anomaly detection on the new metrics? Ask it every time, and do not hand off to document until it has been asked and answered. A no ends the step in one line. An ad hoc analysis skips the offer entirely; nothing recurring lands, so there is nothing for a test to guard.

On a yes:

1. Scan the validated models for their measures and their time axis. Candidates are the metrics a stakeholder would act on if they moved: totals, counts, rates at the model's grain.
2. Before proposing anything, interview the AE about the legitimate variability of the business process behind each metric. The data shows what happened; only the business knows what is normal. Ask the questions that change the design and wait for the answers: are there rare-but-legitimate extreme values (a $3,000 enterprise payment among $30 to $50 consumer payments is real revenue, not an anomaly)? Does the process have known rhythm or seasonality (end-of-month billing pushes, quiet holidays)? Are there known one-time events in the history (a price change, a migration, a backfill)? Which movement would someone actually act on: volume, dollars, or both? The answers shape the grain, the mechanism, and the threshold: legitimate rare values wash out at an aggregate grain and trip tests at a fine one. A candidate that would fire on behavior the AE just described as normal is mis-grained or mis-thresholded and gets reshaped or dropped before it is ever proposed.
3. Propose two to four targeted candidates, never a blanket battery, each as a decision block: the metric, the mechanism (chosen per [references/anomaly-tests.md](references/anomaly-tests.md)), the threshold with its reason, the real failure it would catch, and the false-positive risk on known data quirks (partial edge periods, known step changes, the legitimate extremes from the step above). Wait for the requester's pick.
4. Implement each chosen test as a singular test in `dbt/tests/` following the reference's SQL shapes: thresholds as jinja `set` literals at the top, partial edge periods excluded, a minimum-prior-periods guard, scoring restricted to the latest complete period, an empty `accepted_periods` ruling ledger ready for future rulings, and a stated severity (`warn` unless a firing should stop the pipeline). Run it with `dbtw test --select <test_name>`, named explicitly.
5. A test that fires on known history is a conversation, not a commit: either the flagged period is a genuine finding (report it before shipping anything) or the threshold gets tuned above a known, explainable change with the reason stated in the test's comment. Once shipped, a future firing follows the remediation path in the reference (diagnose the data first, then accept and absorb, redesign, or retire), so an anomaly that turns out to be normal business behavior always has a stated exit instead of a quietly widened threshold.
6. Tell document what landed, so the guide's "How problems are caught" section covers the anomaly guard in its plain-English inventory.

## Hand off

- A check failed because of the model's logic: back to **build-model** with the failing case.
- A check failed because of a data fact nobody knew: **explore-data** to establish it, then refine the contract with **refine-request** if the fact changes the design.
- All checks pass on a buildout: ask the anomaly coverage question above, then **review** for the conventions and design pass, or **document** if the work is done and needs its guide. Document never starts before the anomaly question is asked and answered.
- All checks pass on an analysis: the answer is now final; back to **quick-query** to state it, then **document** writes the analysis doc for anything kept.
