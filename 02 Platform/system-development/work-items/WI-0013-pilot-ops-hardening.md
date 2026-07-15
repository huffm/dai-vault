---
title: "WI-0013 Pilot Operations Hardening v1"
type: "plan"
date: "2026-07-14"
status: "complete"
project: "DAI"
slice: "WI-0013 Pilot Operations Hardening v1"
repos:
  dai: "code (duplicate creation guard + metering coverage; branch wi/0013-pilot-ops-hardening)"
  dai-vault: "docs (runbook, ledger templates, RC drill package, this WI, MOC, current-slice, handoff)"
tags:
  - system-development
  - work-item
  - operations
  - release
related:
  - "04 Products/sports-v1/v1-release-definition-and-scope-freeze-v1.md"
  - "06 Execution/plans/v1-release-critical-path-2026-07-14-v1.md"
  - "06 Execution/plans/v1-concierge-operations-runbook-v1.md"
  - "06 Execution/plans/v1-rc-drill-package-v1.md"
---

# wi-0013 pilot operations hardening v1

**Slice type:** production behavior change (creation guard) + telemetry correction
(metering coverage) + operational doctrine (runbook/ledger/drill package).
**Opened:** 2026-07-14. Third and final item of the frozen V1 critical path.
**This WI does NOT authorize the RC drill, paid calls, or reconciliation writes** --
those are separately gated (see the drill package's four authorization gates).

## problem (three demonstrated operational gaps)

A. Duplicate active-run creation was unguarded at UI and API (2026-07-10 day-2 capture
   discipline was manual) -- an accidental second POST double-spends and leaves an
   ambiguous operating state.
B. The metering price table covered only gpt-4o-mini while the AgentProfile default is
   gpt-4.1-mini -- a configured model metered as a silent None cost.
C. The daily concierge workflow and its recovery procedures lived in session history and
   scattered records, not one executable operator runbook.

## decision (guard semantics)

Fail-closed 409 Conflict at creation, no public override (an override is a new
authorization surface without a demonstrated V1 need). Identity precedence: known gamePk
(request GamePk / resolved ExternalGameId / pending row's own InputJson GamePk) compares
by EQUALITY -- two different gamePks for the same teams and date stay independently
creatable (doubleheader doctrine; order never inferred). When either side lacks a known
gamePk, fail closed on the canonical matchup identity (tenant + competition + gameDate
query scope; normalized home + away): a deliberate second same-matchup capture must carry
an explicit distinct gamePk.

Status matrix: excluded (any status) never blocks; failed never blocks (retry allowed);
pending blocks (concurrent duplicate); completed non-excluded blocks; other tenants never
block or leak; identity-less legacy rows block only via their own persisted matchup;
rows with no matchup evidence never block.

Timing: the check runs BEFORE the pending row and BEFORE any paid work, inside a
per-matchup-identity in-process gate ([check + insert] serialized), so concurrent
duplicates cannot both pass. Single-operator-host assumption documented; a db uniqueness
constraint is the excluded-schema multi-instance follow-up. Reconciliation and settlement
matching are untouched (historical multiple runs per game remain supported).

## metering decision

PRICING covers every currently-configurable model (gpt-4o-mini analyzer constant +
gpt-4.1-mini AgentProfile default; openai public list 2026-07-14: 0.40/1.60 per 1M).
`estimate_cost` returns an explicit `pricingStatus` (priced | unpriced_model |
tokens_unavailable); an unpriced model emits a loud structured cost-log WARNING with the
model id and request id while preserving tokens/latency/request identity. No guessed
fallback price; no request failure; telemetry only -- stripe remains revenue truth.

## scope

1. `DuplicateRunGuard.cs` (pure classifier + per-identity creation gates) + Create-flow
   integration in `AgentRunsController` (409 with minimum safe correlation info).
2. `model_metering.py` pricing coverage + pricingStatus + analyzer unpriced warning.
3. Vault: `v1-concierge-operations-runbook-v1` (full daily workflow + R1-R14 bounded
   recovery), `v1-delivery-ledger-template-v1` (delivery ledger + operator-time log,
   privacy rules), `v1-rc-drill-package-v1` (prepared, NOT executed; four separate
   authorization gates; caps 2 calls / 2 runs; criteria mapped to the freeze doc).

## non-goals / exclusions

RC drill execution; scheduler/automation; email automation; Stripe code; deployment;
auth changes; history page; aggregate dashboards; /metrics changes; registry default;
prompt/model-selection/scoring/confidence/calibration changes; settlement/reconciliation
behavior changes; schema migration (explicitly excluded -- concurrency handled without
one); identity-status; WI-0002/0003; doubleheader capture operation; push/merge
(integration separately gated).

## validation record (2026-07-14)

- red-first throughout: metering (5 red -> 11/11 green), guard (compile red -> 20 green),
  and the review-driven robustness fixes (2 red -> green).
- suites: DevCore.Api.Tests **1235/1235** (baseline 1212, +23: 14 guard unit + 9 guard
  integration); agent-service pytest **453/453** (+5 metering tests); sports-app vitest
  **134/134** (regression only; no Angular production change; bundle compiles). No
  existing test tripped the guard (multi-run tests seed via db or use distinct inputs);
  WI-0011/0012 buyer contracts and all reconciliation/settlement suites unchanged and
  green.
- guard behavior proven by tests: first-proceeds/second-409 with ZERO service invocations
  and ZERO new rows on rejection; distinct doubleheader gamePks both creatable; excluded
  and failed rows never block; pending blocks a concurrent duplicate; cross-tenant rows
  never block or leak; gamePk<=0 keeps the 400; two SIMULTANEOUS identical requests yield
  exactly one run + one 409 + one service invocation (per-identity in-process gate).
- focused review (2 angles): code -- 5 fixed (orientation-insensitive matchup + gate key;
  punctuation/whitespace-tolerant normalization; collation-independent competition
  comparison in memory; date dropped from the gate key so the semaphore dictionary is
  bounded by real team pairs; deterministic oldest-first 409 correlation id), 1 refuted
  (non-numeric provider ids are deliberately unknown identity -> fail closed, now
  unit-tested and documented). docs -- 7 fixed (agent-service liveness is /api/ping; full
  calibration-rows path; R1 known-gamePk precondition; PII kept to the private
  alias-mapping note; $/operator-hour metric aligned to the freeze doc with double-count
  rule; drill criterion 7 re-mapped to the source-outage class; criteria labels
  de-duplicated), 1 refuted (/reconcile does return literal MatchKind names).
- documented limitation: team-name alias canonicalization is out of guard scope -- the v1
  workflow sources names from one screened list; the guard is fail-closed for that
  consistent-naming workflow. multi-instance hosting needs a db uniqueness constraint
  (schema; excluded here) -- the named follow-up.
- rc acceptance correction confirmed already present (freeze doc criterion 1 carries the
  WI-0011-era test-transaction dry-run wording; no stale contradictory record found).
- live verification: not repeated -- every referenced route/script in the runbook was
  verified against source by the docs review, and each procedure (cold start, health,
  GET surfaces, WI-0004 shutdown, cache-restart) was exercised live earlier in this
  operating session; pricing configuration verified by tests without a model call.
- **the RC drill was NOT executed.** the package is prepared; its four authorization
  gates (pregame drill / settlement / payment-link test / no real buyer delivery) remain
  closed.

## links

- work item: WI-0013
- branch: `wi/0013-pilot-ops-hardening` (dai, from `7152818`)
- pr: -- (not authorized)
- commits: dai `85a8831` (7 files, +711/-27; guard + metering + tests + review fixes,
  LOCAL ONLY) + dai-vault docs commit (runbook, ledger templates, drill package, this
  WI, MOC, current-slice, handoff)
- tests: DevCore.Api.Tests 1235/1235; agent-service pytest 453/453; vitest 134/134
- docs updated: this WI; MOC; `06 Execution/plans/v1-concierge-operations-runbook-v1`;
  `06 Execution/plans/v1-delivery-ledger-template-v1`;
  `06 Execution/plans/v1-rc-drill-package-v1`; current-slice; handoff

## final disposition

Implementation complete, local only (2026-07-14). Review resolved (12 findings fixed,
2 refuted). Integration and push separately gated; the RC drill separately gated and NOT
executed.
