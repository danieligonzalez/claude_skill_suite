---
name: refine-request
description: Frame a new modeling or SQL request before any code exists. Use when the user or a stakeholder states a task, business question, or metric ask. Routes the lane first (one-off analysis to quick-query, buildout stays here, asking when ambiguous), then states the output grain, asks only the clarifying questions that change the answer, names the SQL pattern, and decides where the work lands in the DAG. Entry point of the skill chain.
---

# refine-request

Frame the work before writing any SQL. The output is a short contract the rest of the chain executes against. This skill writes no queries and no models.

## Steps

Before framing anything, identify the warehouse and the runtime (CONVENTIONS.md, "Warehouse and runtime"): the warehouse decides the cost model and dialect ([../shared/warehouses/README.md](../shared/warehouses/README.md)), the runtime, dbt Core or dbt Platform, decides how models get discovered and queries run ([../shared/runtime.md](../shared/runtime.md)). Say both out loud when a stakeholder is watching, the same way the lane gets routed out loud below.

1. **Anchor the ask.** Zoom out before anything technical. Who is asking, what decision does the answer drive, and what will they do differently once they have it. Then map every noun in the ask to a real table or column before going further. Start from [../shared/warehouse-map.md](../shared/warehouse-map.md), which holds the current DAG surface on one page; open model files only for what the map does not answer. On dbt Platform, the MCP discovery tools (`get_all_models`, `get_node_details`, `get_all_sources`) are the source of truth instead: the discovery API is the live map, so query it rather than grepping or opening files. A noun with no home (an ask about services in a warehouse with no services table) is the biggest finding in the request; it becomes the first clarifying question, never a silent substitution.
2. **Route the lane, out loud.** Before framing anything: is this a one-off analysis or a model buildout? The test is consumption (quick-query skill): looked at once by the asker is an analysis; checked repeatedly, refreshed, or trusted by others is a buildout. The test is consumption and nothing else. What the warehouse currently holds is not a routing signal: a question with no fact table behind it is still a question, and the missing table means the analysis may take more queries, never that the requester asked for a model. Inferring a buildout from what the DAG lacks answers a question the requester never asked. When the ask leaves consumption ambiguous, that is the first question to the requester ("is this a one-time look, or something you will check again?"), and when it is obvious, the routing still gets said in one sentence so the requester hears the calibration. An analysis routes to quick-query and this skill stops; a buildout continues below. No model file ever lands from the analysis lane.
3. **Grain.** State the output contract in one sentence: one row per ___, with columns ___. This is the correctness spec and the query shape; the final CTE is almost always a group by at exactly this grain. If the grain is unclear, that is a clarifying question.
4. **Clarifying questions.** Ask the ones that change the design, request level before technical: purpose and population first (is this the measure they need, who counts), then boundary definitions, null semantics, tie handling, which statuses count, calendar versus rolling windows. Ask as many as change the answer, batched in one pass; four sharp questions beat two vague ones, but a question whose answer would not change the SQL is padding. Questions the data itself can answer are not clarifying questions; those go to explore-data.

   Format each question as a decision block so the requester can answer with a letter:

   > 1. What counts as a service?
   >    There is no services table; payments carry a service_date.
   >    a) Treat a payment's service_date as the service (my default: it is the only record we have)
   >    b) Wait for a true services source
   >    Why it matters: option a makes unbilled services invisible.

   The question, the candidate readings with a default marked, one line on why it changes the answer. The block shows the ambiguity was seen and hands the requester a one-word way to resolve it.

   Ask means ask. When a requester is present (a stakeholder, including when working under time pressure), the questions go to that person and the skill stops until they answer. Asking a sharp boundary question out loud is part of the deliverable; it shows the ambiguity was seen. Only when nobody is available to answer do you pick the most defensible reading yourself, and then the choice travels as a stated caveat through validate and document, never silently.
5. **Name the pattern.** Match the ask against [references/patterns.md](references/patterns.md). If it maps, the idiom is the answer; plan to instantiate it cleanly. Inventing custom machinery (end labels, hashes, singleton special cases) is the alarm that a pattern was missed.
6. **Decide where it lands.** If the project has a semantic layer and the ask is a metric that already exists, check `list_metrics` / `query_metrics` before framing a new rollup: a metric that exists is not a model to rebuild. A question worth answering more than once is a model, not a query (CONVENTIONS.md rule). Walk the landing rules in [references/worked-example.md](references/worked-example.md) top to bottom, first match wins. Before any SQL, state the DAG addition:
   - Which layer. Does staging already provide the inputs? Does an intermediate model earn its place, or does the mart read staging directly?
   - The model's name and grain, following the conventions in CONVENTIONS.md (`fct_`, `dim_`, `int_`).
   - The materialization, with its reason (CONVENTIONS.md materialization rules): incremental when the fact qualifies, table when it does not, view only for a light transformation. Saying it here shows the production build was designed, not defaulted.
   - The Kimball type (CONVENTIONS.md model conventions): transaction fact, periodic snapshot, accumulating snapshot, dimension, SCD Type 2, or bridge. Saying the type is saying the grain and the update behavior in one word.
   - What the model makes reusable that an ad hoc query would throw away: the tested grain, the documented columns, the next question that becomes a select instead of a rebuild.
7. **Present the plan and get the go ahead.** State the framing block and stop. The check-in happens even when there are no open questions: the plan itself is a decision the AE running the session owns, and when working under time pressure the stakeholder hears it too. Take any input, restate what changed, and hand to build-model only on an explicit go ahead. This is one sentence and a pause ("that is the plan; anything you would change before I build?"), not a ceremony.

## Output

A framing block, stated in conversation before any code. It is a proposal, not an announcement; building waits for the go ahead in step 7.

- Deliverable: what the requester gets, and the decision it feeds.
- Grain: one row per ___.
- Type: the Kimball type of each model added.
- Pattern: the named pattern, or "no catalog match" with the plan.
- Lands as: model name, layer, materialization with its reason, and what it reads from.
- Open questions: only the ones that change the design, as decision blocks. Put them to the requester and wait for answers before building; self-answered caveats are the fallback, not the default.

A full worked example of the steps and the framing block is in [references/worked-example.md](references/worked-example.md). Match its shape every time.

## Hand off

- The ask is a one-off query, consumed once by the person asking: **quick-query**, not this chain. The second ask of the same question comes back here.
- Source facts unknown (key uniqueness at the claimed grain, duplicates, orphans, value domains, date coverage): invoke **explore-data** first, then finish the framing with its brief.
- Framing presented and the go ahead given: hand to **build-model**. Building does not start inside this skill, and never starts on a plan nobody approved.
- A later finding (from explore-data, build, or validate) changes the deliverable, the grain, or what the requester most likely wants: re-enter this skill and restate the framing block before building continues. A finding that only tightens a definition travels as a stated caveat instead. The test: if the requester would change their ask on hearing it, re-frame; if they would just want it noted, caveat.
