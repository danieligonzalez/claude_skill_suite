---
name: document
description: "Write the documentation for finished work. Use when a model has passed validate and review, when a kept ad hoc analysis needs its write-up, or when docs have drifted from code. For a buildout: yml descriptions in the folder scaffold plus a data guide in docs/. For an ad hoc analysis: a short analysis doc in docs/ that outlines the results and how they were arrived at."
---

# document

Explain the work so the next person (or the next session) uses it instead of rebuilding it. The artifact matches the work. A buildout gets two artifacts with different jobs: yml descriptions answer "what is this column", the data guide answers "what is this pipeline and can I trust it". A kept ad hoc analysis gets one small artifact, the analysis doc, which answers "what did we learn and how". Know which is being written before writing anything.

## yml descriptions

Land in the folder's `_<folder>__models.yml`. The CONVENTIONS.md rule: concise, what the field holds and how it connects to the purpose of the table. Never restate the column name; `payment_id: "The payment id"` is a missing description with extra steps. The dbt bundle's `writing-documentation` reference inside `using-dbt-for-analytics-engineering` covers the craft; use it instead of inventing style here.

## The data guide

One markdown file per mart or pipeline in `docs/`, short enough to read in two minutes. The reader is a stakeholder, not an engineer: they should finish it knowing what the numbers mean, how far to trust them, and what to do next. Write to that reader. Plain declarative sentences that state facts without hedging or justifying. Every technical term either earns its place or gets a plain gloss the first time it appears; "grain" can stay once defined, "left-censored" needs one clause of English beside it. If a sentence only makes sense to the person who built the model, rewrite it. Follow [references/guide-template.md](references/guide-template.md); match its shape every time. Structure:

1. **What it answers.** The business questions this model exists for, one or two lines.
2. **The pipeline.** The walk from source to final: each step, what it does to the data, and why it is there. Name the models so the reader can follow along in the DAG. This is prose, not a diagram dump.
3. **Grain and guarantees.** The grain in one sentence, then the guarantees as facts backed by tests: "payment_id is unique and never null (tested)", "every payment joins a customer (relationships test)". A guarantee without a test is a wish; if one is stated here untested, that is a finding.
4. **How problems are caught.** The proactive quality story, written for a VP or C-suite reader with no SQL: every update runs automated checks before the numbers land anywhere, and this section says what those checks watch for in plain English. Cover the categories at the level of what they protect, never test names: the table keeps exactly one row per thing it counts, records always trace back to real entities in the other tables, and the calculation logic is proven against hand-built examples with known answers. Then say what happens on a failure: the update stops, the previous numbers stand, and an engineer gets the alert, so a data problem becomes a delay instead of a wrong number in a meeting. If runs and check results are recorded (Elementary does this here), one line on that: quiet changes surface as alerts, not silent shifts. The test for this section: the reader could repeat it in a meeting to answer "how do we know these numbers are right".
5. **Expected results.** Invariants over exact numbers: magnitudes, known exclusions and why, what a healthy build looks like. Exact counts go stale on the next ingestion; write "12.3k payments as of the 2026-08 load" not "12,352 rows". Flag partial periods at the edges: a first or last period the data only partly covers reads as a real dip unless the guide says otherwise.
6. **How to use it.** One example select answering a real question at the model's grain, in the repo SQL style. When the mart aggregates a row level flag, also point the reader at where the flagged rows live (the model and the flag column), so the guide answers "which rows do I act on", not only the totals.

## The analysis doc

A kept ad hoc analysis gets a smaller, different artifact than a pipeline. One markdown file in `docs/`, named after the analyses file, readable in one minute. It outlines the results and explains how they were arrived at, nothing else. The buildout guide sections above do not apply; brevity is the point.

1. **The question and the answer.** The question as framed, then the findings as plain declarative sentences, each number with its definition attached.
2. **How the results were arrived at.** A brief high level overview of the logic in prose: the sources read, the filters and joins that shape the population, the classification or aggregation that produces the numbers. The reader should be able to predict roughly what the SQL does without opening it.
3. **Caveats.** Sample sizes, partial windows at the edges, and any definition that could reasonably have gone the other way.
4. **Where to rerun.** The analyses file by name and its `dbtw show --select` command, so the numbers can be refreshed on a later load.

No yml lands (analyses have no yml scaffold) and no "How problems are caught" section (no tests guard an analysis; the validate statement in conversation was the proof). A tossed one-off gets no doc; its answer lives in the conversation.

## Rules

- DRY across the two artifacts: column detail lives in yml, the guide links to it and never repeats the column list.
- Before finishing, confirm the models covered by the guide have current rows in [../shared/warehouse-map.md](../shared/warehouse-map.md) (the upkeep rule lives there; build-model maintains it, this is the backstop).
- Plain declarative prose, no em-dashes (CONVENTIONS.md rule).
- Concise beats complete. A guide nobody reads documents nothing.
- The test before shipping: could the stakeholder who asked the original question read this, understand it, and act on it without a follow-up meeting? If any section needs the author in the room to explain it, that section is not done.

## Hand off

- Writing the guide surfaced a design smell (a step that cannot be explained simply usually should not exist): **review**.
- A stated guarantee has no test behind it: **build-model** to add the test, then finish the guide.
