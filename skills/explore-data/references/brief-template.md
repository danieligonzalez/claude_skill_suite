# Brief template

Every brief in `exploration/` follows this shape, so any session can scan any brief the same way. Values below are illustrative.

```markdown
# payments, profiled 2026-08-20

Source: raw.payments via {{ source('raw', 'payments') }}, 12,352 rows as of the 2026-08-11 load.

Surprises first:
- 41 payments reference a customer_id that does not exist in customers (orphans).
- status has a domain of 5 values including both 'paid' and 'Paid'; casing is not normalized in raw.

The expected facts:
- payment_id unique: 12,352 rows, 12,352 distinct keys. Grain holds.
- Nulls and blanks: amount null in 0 rows; payment_date blank in 12 rows.
- Domains: status = paid (9,801), pending (1,406), failed (720), refunded (384), Paid (41).
- Dates: 2024-01-03 to 2026-08-09, 903 distinct days across a 949 day span.

Implications for the build:
- Staging must lowercase status and decide where the 41 orphans and 12 blank dates land; neither can be silently dropped.
```

Rules for filling it:

- Every line is a fact with its number. "Some duplicates exist" is not a finding; "217 duplicate keys, all identical copies" is.
- Surprises first means anything that would change the framing or that the requester would want interrupted for. If there are none, say "No surprises." and move on.
- The expected facts section keeps the checks that came back clean, one line each, so the next session knows they were run and does not re-run them.
- Implications are one or two lines and only about what the build must do differently. Analysis beyond that belongs in the framing conversation, not the brief.
- Name the file after the table (`payments.md`) or the topic when it spans tables (`payment-customer-joins.md`).
