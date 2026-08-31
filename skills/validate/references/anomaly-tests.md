# Anomaly test mechanisms

How to pick a mechanism and how to write the test. Anomaly tests guard a metric's future behavior, so they are proposed only through the anomaly coverage step in SKILL.md: targeted, few, each defending a metric someone would act on if it moved.

## Choosing the mechanism

The sample size of the time axis decides, not preference:

| time buckets available | honest mechanism | why |
| --- | --- | --- |
| under ~8 (a handful of months) | trailing percent-deviation with an explicit threshold | stddev, MAD, and IQR are all noise at this n; a "3 sigma" test on 6 points is statistical theater. A deliberate percent threshold makes the tolerance a stated business decision instead of a fake statistic. |
| ~20+ daily points with weekday rhythm | day-of-week matched trailing z-score (compare Monday to trailing Mondays) | raw daily stddev conflates weekday seasonality with real anomalies; matching weekday removes the seasonal term before scoring. |
| 7+ buckets and Elementary available | Elementary `volume_anomalies` / `column_anomalies` with `timestamp_column` | with a timestamp column set, Elementary buckets the table's own history in a single run (static data is fine) and z-scores the last buckets against a training window. It hard-errors below 7 training points, which is why monthly grain on a 6-month dataset cannot use it. Its non-timestamp mode needs 7+ real dbt invocations over time and never fits static data. |

## Rules for every anomaly test

- Singular test in `dbt/tests/`, one select returning the anomalous periods as failing rows.
- Thresholds and minimums are jinja `set` literals at the top of the file, one line each, so tuning is a one-character diff.
- Exclude partial edge periods from scoring; they read as anomalies and mean nothing. Keep them in the trailing baseline only when the comment says so and why.
- Guard the cold start: require a minimum number of prior periods (`prior_n`) before scoring, so the first periods never score against an empty baseline.
- Score only the detection window, the latest complete period, never all of history. A period's trailing baseline is frozen forever (it is built only from the periods before it), so a test that re-scores history re-fires on every already-ruled period on every run and can never go green again. History belongs in the baseline, not in the verdict. Widen the window past one period only when late-arriving data can restate closed periods, and say so in the comment.
- An `accepted_periods` jinja list at the top of the file is the ruling ledger in code: periods a human investigated and accepted are excluded from scoring but stay in the baseline (they happened, and the baseline should know). Each entry carries a dated comment stating the cause. Adding a period here is the one-line change that turns a pipeline green after a legitimate firing.
- Severity is a per-metric decision, stated in the test's config block: `error` blocks the build and forces the conversation before anything ships on top of a suspect load; `warn` surfaces the firing without stopping the pipeline. Default to `error` only when acting on the metric during an incident is worse than a paused pipeline; a metric where the response to a firing is "go look" runs as `warn`.
- A new anomaly test that fires on known history is a conversation, not a commit. Either the flagged period is a real finding (report it through validate before shipping anything) or the threshold sits below a known, explainable step change (tune it above, and state the known change in the comment).
- Known step changes and long-window drift: an expanding trailing average absorbs old regimes slowly; a bounded trailing window (last N periods) forgets them. Pick per metric and say which in the comment.
- Drift is the mechanism's blind spot, measured, not guessed: on bench fixtures, 5 percent compounding monthly growth peaked at 29.5 percent deviation against an expanding baseline over 12 months, never crossing a 0.30 threshold. If gradual drift matters for the metric, propose a different guard (period-over-period slope, or same-period-last-year) and say the trailing test does not cover it.
- A permanent step change (a fee increase) fires for one or two periods and is then absorbed by an expanding baseline. The double-fire is expected, and the conversation it forces (real business change versus incident) is the point of the test.
- Legitimate variability comes before thresholds. Ask the business what normal extremes look like for the process (an enterprise customer paying $3,000 among $30 to $50 consumer payments is real revenue, not an anomaly). Rare legitimate values wash out in an aggregate like a monthly total and trip tests at a fine grain like per-payment ranges or daily totals. A test that would fire on known-legitimate behavior is mis-grained, not mis-thresholded: move it up a grain, or set the threshold above the legitimate extreme and state that extreme in the comment.

## When a shipped test fires: the remediation path

A firing anomaly test is a question, not a verdict. Walk this path in order, and leave a written ruling in the test file every time.

1. **Diagnose before touching the test.** Check the flagged period's inputs: row counts by source, a duplicate check on the natural key, the load dates. A data defect (a double load, a missing partition, a join fan-out) means the test did its job; fix the data and change nothing in the test.
2. **One-time legitimate event** (a backfill, a promotion month, a weather week): leave the threshold alone and unblock through the ruling ledger: add the period to the `accepted_periods` list with a dated comment stating the cause. That one-line change is the ruling made executable; the test goes green on the next run and the period stays in the baseline. If the period has already rolled out of the detection window, the test healed itself and the ruling is comment-only. Never widen a threshold for a one-off; that buys silence on the next real incident.
3. **New permanent normal** (a fee increase, a new customer segment): accept and absorb. Expect the one-to-two-period double-fire; unblock each fired period through `accepted_periods` with the ruling recorded (the date, what changed, who confirmed it is business reality) while the expanding baseline catches up. If the metric re-bases often, switch the window choice from expanding to bounded so old regimes are forgotten by design, and state the switch in the comment.
4. **Recurring variability the interview missed** (seasonality, rare legitimate extremes): the test is mis-designed, not noisy. Go back through the legitimate-variability rule: re-grain the test, or set the threshold above the stated extreme with that extreme named in the comment. This is a design change and ships through a PR like any model change.
5. **Nobody would act on the firing:** retire the test. A test whose failures are routinely waved through is worse than no test; it trains everyone to ignore red.

Two invariants hold on every path: a threshold moves only with a stated business reason recorded in the file, never to make the test green, and every accepted firing leaves a dated ruling in the test comment. Over time the comment block becomes the metric's case history, which is exactly what the next person investigating a firing needs first.

## SQL shapes (DuckDB, repo style)

Monthly metric, trailing percent-deviation, expanding window, scored on the latest complete month only:

```sql
{% raw %}{{ config(severity = 'warn') }}

{% set deviation_threshold = 0.30 %}
{% set min_prior_months = 2 %}
{% set accepted_periods = [] %}  -- ruling ledger, e.g. ['2023-02-01']  2023-02: fee increase, confirmed by billing ops{% endraw %}

with monthly_totals as (

    select month_start_date
         , sum(metric) as metric_value
    from {% raw %}{{ ref('the_mart') }}{% endraw %}
    group by month_start_date

)

, latest_month as (

    select max(month_start_date) as max_month_start_date
    from monthly_totals

)

, complete_months as (

    select monthly_totals.month_start_date
         , monthly_totals.metric_value
    from monthly_totals
    cross join latest_month
    where monthly_totals.month_start_date < latest_month.max_month_start_date

)

, trailing_stats as (

    select month_start_date
         , metric_value
         , avg(metric_value) over (
               order by month_start_date
               rows between unbounded preceding and 1 preceding
           ) as trailing_avg
         , count(*) over (
               order by month_start_date
               rows between unbounded preceding and 1 preceding
           ) as prior_month_count
    from complete_months

)

, detection_window as (

    /* score only the latest complete month; earlier months are settled
       history and re-firing on them would block the pipeline forever */
    select max(month_start_date) as scored_month_start_date
    from complete_months

)

select trailing_stats.month_start_date
     , trailing_stats.metric_value
     , round(trailing_stats.trailing_avg, 2) as trailing_avg
     , round(abs(trailing_stats.metric_value - trailing_stats.trailing_avg) / nullif(trailing_stats.trailing_avg, 0), 3) as pct_deviation
from trailing_stats
cross join detection_window
where trailing_stats.month_start_date = detection_window.scored_month_start_date
  and trailing_stats.prior_month_count >= {% raw %}{{ min_prior_months }}{% endraw %}
  and abs(trailing_stats.metric_value - trailing_stats.trailing_avg) / nullif(trailing_stats.trailing_avg, 0) > {% raw %}{{ deviation_threshold }}{% endraw %}
{% raw %}{% if accepted_periods %}
  and trailing_stats.month_start_date not in ({% for period in accepted_periods %}'{{ period }}'::date{% if not loop.last %}, {% endif %}{% endfor %})
{% endif %}{% endraw %}
```

Daily volume, day-of-week matched trailing z-score: bucket by date and weekday, join each day to its prior same-weekday days inside a bounded trailing window (56 days), require `prior_n >= 5`, flag `abs(z) > 3`. Same exclusion and guard rules as above; `stddev_samp` is legitimate here because each weekday accumulates 20+ observations over 6 months.

Sources behind these choices: Elementary anomaly detection docs (method, sensitivity, 7-point floor, seasonality config), dbt-expectations `expect_column_values_to_be_within_n_moving_stdevs`, dbt Labs guidance on reserving singular tests for high-impact business-specific checks.
