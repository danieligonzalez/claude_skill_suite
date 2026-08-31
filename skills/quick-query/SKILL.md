---
name: quick-query
description: Answer a one-off SQL question without building architecture. Use when the requester asks for a query or a number they will consume once, now. One-line frame, queries written into an analyses file as the work happens and run from it by name through dbtw show, a 30 second self-check, the answer with its assumption attached. Never writes models, tests, or yml. The second ask of the same question escalates to refine-request.
---

# quick-query

The one-off lane. The six-skill chain builds assets; this lane answers a question and stops. The routing test takes one second: who consumes the answer, and how often? Consumed once, now, by the person asking: this lane. Reused, refreshed, joined against, or trusted by other people later: refine-request and the chain. Say the routing out loud when a stakeholder is watching; calibrating effort to the ask is part of the answer.

## Steps

1. **Frame in one line.** Restate the question as grain plus definition before any SQL: "top 5 customers by total dollars, all time, both copies of a duplicated payment counted." One decision block only when a fork genuinely changes the answer; otherwise state the default and keep moving.
2. **Open the analysis file before the first query.** Every query whose result feeds the answer is written into a file in `dbt/analyses/` first and run from there by name. One select per file, named for what it computes. The file comes first and the run comes second, so the requester can follow the work as it happens and rerun any cut themselves; a number never appears in conversation from SQL that exists nowhere. The file opens with a three line comment header: the question, the date, the definition used. dbt compiles analyses (`ref()` and jinja work) but never materializes them, so the workbench gets version control without joining the DAG.
3. **Write it in repo style.** CONVENTIONS.md SQL rules apply: leading commas, CTEs, no subqueries, full name table references. Read from `ref()` on staging or marts first, because cleaning already happened there; `source()` only when the question is about the raw data itself, with casts.
4. **Run it by name: `dbtw show --select <file_name>`.** The default preview limit is 5 rows; pass `--limit` when the answer is longer. `dbtw show --inline` is reserved for throwaway shape checks (a row count, a duplicate peek) whose numbers never reach the answer; anything the answer rests on runs from its file.
5. **The 30 second self-check.** One count against expectation, or one boundary row eyeballed, stated in one line. Skipping it turns the answer into a guess.
6. **State the answer with its assumption attached.** The definition from step 1 travels with the number.

## Keep or toss

- The files already exist by the time the answer lands, so keep or toss is a deletion decision made at the end, with the requester.
- Most one-offs get tossed: the files are deleted in the same session and the answer lives in the conversation. That is still the point of the lane.
- A keeper stays in `dbt/analyses/` with its header. For a multi-file analysis, the cut that produced the headline finding is the keeper candidate; supporting probes usually go.
- The second time the same question gets asked, it stops being a one-off: route to **refine-request** and it becomes a model. A question worth answering more than once is a model.

## Boundaries

- Never writes models, tests, or yml, and never materializes anything.
- A question that needs profiling before it can be answered safely (unknown grain, suspected duplicates): **explore-data** first, then answer.

## Hand off

The lane is short but it is still a chain: answer, then proof, then record.

- Answer stated: **validate** runs its analysis mode before the number is treated as final or reported onward. The checks scale to the claim (validate states how); the second path to the headline number is the floor, and for a one-number quick hit that second path is a single recount by a different route.
- Validated and kept: **document** writes the analysis doc, the short form that outlines the findings and how they were arrived at. A tossed one-off skips the doc; its answer lives in the conversation.
- The second ask of the same question: **refine-request**, and it becomes a model.
