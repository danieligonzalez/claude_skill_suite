---
name: review
description: Review a model, a set of models, or a design against the repo conventions and architecture. Use after validate passes, at the end of a work stretch, or on request. Judges design and sustainability, not numbers. Produces ranked findings; fixes are a separate step.
---

# review

The design pass. validate proved the numbers; review judges whether the work will hold up: conventions, architecture, and what happens when the data grows and the next question arrives. This skill reports findings and changes nothing; fixes go back through build-model as their own step.

## The checklist

Walk each area against the model or design under review:

1. **Conventions.** CONVENTIONS.md compliance: SQL style, naming, folder placement, yml entry present with descriptions and tests. One line per miss.
2. **DRY.** Logic that staging already owns re-applied downstream, an expression appearing in two models without a macro, literals that should be vars, a helper dbt_utils already provides.
3. **Grain and joins.** Is the grain stated and tested? Any one to many join before an aggregate (fan out that silently inflates numbers)? Any join on a column whose uniqueness was never proven?
4. **Test coverage, both directions.** The grain key has unique and not_null. Keys arriving from outside the model have relationships. Complex derivations, date windows above all, have a unit test. Name the gap, not just "add more tests". Then check the opposite: tests that defend nothing are findings too. accepted_values on a case statement's hardcoded outputs, not_null on a coalesced column, an assertion true only because today's data happens to satisfy it. Every test is a production query with run time and warehouse cost; a test that cannot fail, or that encodes a coincidence instead of an invariant, gets removed, not admired.
5. **Sustainability.** What breaks when next month's data arrives: is the incremental logic idempotent, do filters hardcode dates that should be vars, will a new status value pass silently or fail loudly? Is the next likely question a select against this model or a rebuild of it?
6. **Placement.** Does the model sit in the right layer, does it duplicate an existing model's job, should any of it move upstream so other models can share it? Does the stated Kimball type match what the model actually is? Does the materialization follow the CONVENTIONS.md rules: an incremental fact that passed its proof sequence, a table where incremental does not qualify, no view doing heavy work?
7. **Fit for purpose.** validate proved the number is right; this check asks whether it is useful. A metric that barely discriminates (nearly every row shares one value), a rate over a denominator too small to mean anything, a measure the requester cannot act on. A correct number that answers the question poorly is a finding: say so and propose the sharper measure.

## Output

Findings ranked in three tiers, stated in conversation:

- **Blocking**: wrong numbers possible or conventions broken. Fix before shipping.
- **Should fix**: will cost real time later, fine to ship behind a note.
- **Note**: worth knowing, no action required.

Each finding: what, where (model and line if it is code), why it matters, and the suggested fix in one line. A clean review says so plainly and states what was checked. Follow [references/findings-template.md](references/findings-template.md); match its shape every time.

## Sign off

The review ends with the AE's hands on the data, not with a report. After the findings:

1. Provide two to four validation queries the AE runs manually, each with one line stating what it should show ("one row per month, 65 to 70 rows", "returns zero rows"). Pick them to let the AE independently confirm the grain, the headline number, and any finding a query can demonstrate. They are for a human in the DuckDB UI, so they use plain schema-qualified SQL (`main_marts.fct_x`), not `dbtw show --inline` and not jinja; the dbtw rule governs the agent's queries, and these are not the agent's queries.
2. Ask for sign off and stop. Blocking findings route to build-model first; sign off is on work the AE would ship.
3. Only after the sign off arrives does the chain move to document. No sign off, no guide.

## Hand off

- Findings to fix: **build-model**, one logical change at a time.
- A finding that questions a number: **validate** to prove it either way.
- Review is clean or findings are accepted, the validation queries are out, and the AE has signed off: **document**. The sign off is the gate; document never starts without it.
