---
title: "V1 Delivery Ledger and Operator-Time Log Templates v1"
type: "plan"
date: "2026-07-14"
status: "active"
project: "DAI"
slice: "WI-0013 Pilot Operations Hardening v1"
repos:
  dai: "unchanged"
  dai-vault: "docs-only"
tags:
  - operations
  - ledger
  - release
related:
  - "06 Execution/plans/v1-concierge-operations-runbook-v1.md"
---

# v1 delivery ledger and operator-time log templates

Plain-file V1 ledgers -- no database tables, no dashboards. The live ledger files are kept
OUTSIDE the vault repo (they will carry buyer identifiers); this document is the template
and the rules. Never commit real customer personal data to the vault; the buyer/pilot
identifier in a committed artifact is always an opaque alias (e.g. `pilot-01`), never a
name or email.

## delivery ledger (one row per promised deliverable)

CSV header (copy verbatim; one file per month, e.g. `delivery-ledger-2026-08.csv`):

```csv
date,pilot_id,entitlement_status,gamepk,run_id,brief_generated_utc,brief_delivered_utc,recap_delivered_utc,delivery_status,failure_or_retry_note,model_calls,external_source_calls,measured_cost_usd,operator_minutes,manual_interventions,buyer_feedback_note
```

Field rules:

- `pilot_id` -- opaque alias mapped to the real buyer in a private, uncommitted note.
- `entitlement_status` -- `paid` (verified Stripe receipt) | `test` (RC dry-run, marked
  test transaction) | `withheld` (delivery blocked, see runbook section 7).
- `run_id` -- the AgentRunId; `gamepk` -- the explicit provider game id.
- timestamps UTC ISO-8601; blank = did not happen (never invent).
- `delivery_status` -- `delivered` | `late` | `failed` | `withheld` | `skipped`.
- `model_calls` / `external_source_calls` -- actual counts for THIS row's run(s),
  from the cost log and screening notes.
- `measured_cost_usd` -- summed `estimatedTotalCost` from the cost log lines
  (pricingStatus must be `priced`; an unpriced line is a runbook R3 stop).
- `manual_interventions` -- count of runbook recovery paths exercised.

Markdown mirror (for quick daily review inside a private note, same columns):

```markdown
| date | pilot | entitlement | gamePk | run | brief gen | brief sent | recap sent | status | note | calls | ext | cost | min | interv | feedback |
|------|-------|-------------|--------|-----|-----------|------------|------------|--------|------|-------|-----|------|-----|--------|----------|
```

## operator-time log (one row per operating day)

```csv
date,slate_screened,runs_created,deliveries,operator_minutes_total,minutes_opening,minutes_screening,minutes_generation,minutes_delivery,minutes_settlement,minutes_recovery,notes
```

Rules: minutes are wall-clock honest estimates recorded the same day. The pilot metric
"revenue per unit of operator attention" is computed exactly as the freeze doc defines it:
**dollars per operator-HOUR across the whole pilot** = pilot revenue / (sum of
`operator_minutes_total` across all operating days / 60). The per-day
`operator_minutes_total` in THIS log is the authoritative input; the per-row
`operator_minutes` in the delivery ledger is a diagnostic allocation only and is never
summed into the metric (it would double-count). Target trend (freeze doc): <= 30 minutes
per delivery day by pilot week 2.

## retention

Keep ledgers for the full pilot; the 2026-08-21 evaluation reads them directly. Weekly:
copy an ANONYMIZED aggregate row (counts + cost + minutes only) into the pilot evaluation
note in the vault.
