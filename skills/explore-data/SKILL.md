---
name: explore-data
description: "Profile source or model data before building on it. Use when source facts are unknown: key uniqueness at a claimed grain, duplicates, orphaned records, null rates, value domains, date coverage. Produces a short written brief in exploration/. Profiling only; never computes the requested metric."
---

# explore-data

Establish the facts about the data before anything is built on it. The output is a brief, not an answer. This skill answers "what is this data like", never "what is the number".

## Rules

- Aggregates only. Never dump rows. A sample is at most 5 rows, and only when the shape of a value matters.
- Every query runs through `dbtw show --inline` with `{{ source() }}` or `{{ ref() }}` (CONVENTIONS.md rule).
- The moment a profiling query starts computing the requested metric, stop. That is the build, and it belongs to build-model after refine-request has framed it.
- Briefs live in `exploration/` at the repo root, a sibling of `dbt/`, never inside it. Check there first. If a brief already covers the table, read it instead of re-profiling. Re-run a check only if the data has been re-ingested since the brief was written.

## The profile

Run what the framing needs, not all of it every time:

1. Row count and key uniqueness at the claimed grain: `count(*)` against `count(distinct key)`.
2. Duplicate keys: how many, whether the copies are identical or conflicting, and how the copies sit in time (same day, or spread across days). The spacing is often the clue to the mechanism that made them, and it belongs in the brief whether or not anyone asked.
3. Orphans: anti-join across the claimed relationship, both directions.
4. Null rates on the columns the work depends on.
5. Value domains on low cardinality columns: status, type, category.
6. Date coverage: min, max, and gaps when the pattern is time based. Raw columns are varchar, so cast first.

Canonical query shapes are in [../shared/query-shapes.md](../shared/query-shapes.md), profiling section.

## Output

A brief at `exploration/<table or topic>.md` in the repo root folder (the rules above pin the location), committed with the work it supported. One line per finding, stated as a fact with its number. Surprises first. End with one or two lines of implications for the build. Follow [references/brief-template.md](references/brief-template.md) exactly so every brief scans the same way.

## Delegation

The profile is well defined work: hand it to a sonnet subagent. The manager session decides which checks the framing needs; the subagent runs them through `dbtw show --inline` and returns one line per finding with its number, plus a draft brief. The manager reviews the numbers, writes the implications, and commits the brief. Judgment stays with the manager; so does anything surprising. Subagents run one at a time (CONVENTIONS.md delegation rule): profile tables serially, never one agent per table in parallel. When working under time pressure, skip the hand off and run the battery inline; the checks are identical either way (CONVENTIONS.md pacing rule).

## Hand off

- A finding changes the ask (the claimed grain does not hold, orphans exist, a domain is dirtier than assumed): say so in conversation before anything is built. This is the finding the requester needs to hear first.
- Framing was waiting on these facts: return to **refine-request** to finish the contract.
- Framing already agreed and the facts came back clean: hand to **build-model**.
