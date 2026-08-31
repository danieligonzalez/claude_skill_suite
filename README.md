# claude_skill_suite

An opinionated set of Claude Code skills for **analytics engineering with dbt**. Together they turn a vague data ask into a framed, built, tested, validated, and documented dbt model, one hand-off at a time. A separate one-off lane answers throwaway questions without building architecture.

The skills were extracted from a working dbt + DuckDB project and generalized. Every example uses a neutral `customers` + `payments` warehouse; swap in your own objects.

## The two lanes

**The chain** (build a durable, tested asset):

| Skill | Job | Hands off to |
|---|---|---|
| `refine-request` | Frame the ask: route the lane, state the grain, ask only the questions that change the answer, name the pattern, decide where it lands in the DAG | explore-data, build-model |
| `explore-data` | Profile the sources (grain, duplicates, orphans, nulls, domains, coverage), write a brief to `exploration/` | refine-request, build-model |
| `build-model` | Sketch the CTE chain backward from the grain, write the model with its yml docs and tests, build green | validate |
| `validate` | Counts walk, boundary rows, a second path to the headline number, a traps scan, an anomaly-coverage offer | build-model, review, document |
| `review` | Ranked findings on design and sustainability, then hands the analyst validation queries to run | build-model, validate, document |
| `document` | yml descriptions plus a stakeholder-readable data guide in `docs/` | review, build-model |

**The one-off lane** (answer once and stop):

| Skill | Job |
|---|---|
| `quick-query` | One-line frame, repo-style SQL written into an analyses file and run by name, a 30-second self-check, the answer with its assumption attached. The second ask of the same question escalates to `refine-request` and becomes a model. |

The routing test is consumption: an answer looked at once by the asker stays in the one-off lane; anything reused, refreshed, or trusted by others goes through the chain.

## What's in here

```
CONVENTIONS.md      The doctrine the skills reference: SQL style, DRY, model and
                    materialization conventions, the database lock, delegation.
skills/
  README.md         The skills index and the dbt bundle it leans on.
  refine-request/   explore-data/   build-model/
  validate/         review/         document/   quick-query/
  shared/           Cross-skill references: warehouse map, query shapes, dataset intake.
```

Each skill is a `SKILL.md` plus a `references/` folder of templates and worked examples. The skills reference `CONVENTIONS.md` by name for the rules they share, so the doctrine lives in exactly one place.

## Using it

1. Copy the seven skill folders from `skills/` into your project's `.claude/skills/` (drop the `shared/` folder in alongside them; the relative links between skills expect it there).
2. Fold `CONVENTIONS.md` into your project's `CLAUDE.md`, or keep it as its own file and adjust the naming to your stack. The skills cite it as `CONVENTIONS.md`.
3. Replace `skills/shared/warehouse-map.md` with a map of your own objects. The example map's shape is the point, not its rows.

### Assumptions the examples make

- **dbt core** for transformations, run through a thin wrapper named `dbtw` (in the source project it hands off a DuckDB UI lock; substitute plain `dbt` if you do not need that). All dbt selections name models explicitly, never graph operators.
- **DuckDB** as the local warehouse, with schemas `raw`, `main_staging`, `main_marts`. Dialect notes call out where DuckDB and Snowflake agree or differ.
- **dbt_utils** installed, and the `dbt@dbt-agent-marketplace` skills bundle available for dbt mechanics (unit tests, command syntax, debugging). `skills/README.md` lists which bundle skills the chain uses.
- **Elementary** (optional) for run and test-result history, referenced in the anomaly-detection and documentation guidance.

None of these are load-bearing for the method. The value is the workflow: frame before building, profile before trusting, test the invariant once at the layer that owns it, prove the number a second way before it leaves the room, and document for the person who will act on it.

## License

Personal project. Use freely.
