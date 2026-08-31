# Validation statement template

The statement said in conversation after the checks run. Same shape every time, so a pass and a fail are equally recognizable. Values below continue the fct_payments_monthly worked example and are illustrative.

A passing statement:

```
Validation of fct_payments_monthly:

1. Counts walk: raw 12,352 → staging 12,311 (41 orphaned payments quarantined per the brief)
   → mart payment_count sum 12,299 (12 blank payment_date rows land in an unknown-month bucket).
   All losses explained.
2. Boundaries: first month 2024-01, last month 2026-08, both match source date coverage.
   3 zero-payment months present with zeros. The 12 blank-date rows appear in the
   unknown-month bucket, not dropped.
3. Second path: collected total 1,204,558.20 from stg_payments equals 1,204,558.20
   from the mart. Exact.
4. Traps: all clear. (Tie handling and window traps not applicable, no windows.)
5. Idempotency: not applicable, table materialization.

Verdict: defensible. Worth watching: the 41 orphans are quarantined, so any
customer-level rollup will not include them; that exclusion belongs in the data guide.
```

A failing statement leads with the failure and stops:

```
Validation of fct_payments_monthly: FAILED at check 3.

Second path: collected total from stg_payments is 1,204,558.20; the mart says
1,209,377.95. The mart is high by 4,819.75, which equals the sum of the 41
'Paid' (capitalized) rows counted as collected by the mart's case-sensitive
comparison but not by the staging computation.

Stopping here. This routes back to build-model: the status comparison must use
the staging-normalized value. Checks 4 and 5 not run; they can wait for the fix.
```

Rules:

- Numbers in every line. A check without its numbers did not happen.
- The walk states the reason for each loss inline, next to the number it explains.
- A failure is reported before any fix, with the routing named (build-model for logic, explore-data for unknown data facts).
- "Worth watching" is for true passes only: the caveat the requester should hear even though nothing failed.
