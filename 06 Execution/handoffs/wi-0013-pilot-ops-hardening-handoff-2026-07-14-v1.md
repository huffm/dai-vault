---
title: "WI-0013 Pilot Operations Hardening v1 -- slice handoff (2026-07-14)"
type: "handoff"
date: "2026-07-14"
status: "complete"
project: "DAI"
slice: "WI-0013 Pilot Operations Hardening v1"
repos:
  dai: "code (duplicate guard + metering; local branch, not pushed)"
  dai-vault: "docs (runbook, ledgers, drill package, WI, MOC, current-slice, this handoff; local)"
related:
  - "02 Platform/system-development/work-items/WI-0013-pilot-ops-hardening.md"
  - "06 Execution/plans/v1-concierge-operations-runbook-v1.md"
  - "06 Execution/plans/v1-rc-drill-package-v1.md"
---

# wi-0013 slice handoff -- pilot operations hardening v1

**Disposition: IMPLEMENTATION COMPLETE, LOCAL ONLY. Review resolved. Nothing pushed.
The RC drill is PREPARED and NOT EXECUTED.**

Governing WI: WI-0013 (`complete`; full validation record inside the WI). Commits: dai
`85a8831` on `wi/0013-pilot-ops-hardening` (from `7152818`; 7 files, +711/-27);
dai-vault docs commit on main (runbook, ledger templates, drill package, WI, MOC,
current-slice entry, this handoff).

## what shipped

1. **Duplicate active-run creation guard** (`DuplicateRunGuard.cs` + Create-flow
   integration): fail-closed 409 before the pending row and before any paid work;
   identity precedence = known-gamePk equality (request GamePk / resolved ExternalGameId
   / pending row's own InputJson GamePk), falling back fail-closed to the
   orientation-insensitive normalized matchup; excluded and failed rows never block;
   pending and completed non-excluded rows block; distinct doubleheader gamePks stay
   independently creatable; tenant-scoped with no cross-tenant leak; concurrent
   duplicates serialized by a bounded per-matchup in-process gate (single-operator-host
   v1; db uniqueness constraint = the excluded-schema follow-up). Reconciliation and
   settlement matching untouched.
2. **Metering price coverage** (`model_metering.py` + analyzer warning): gpt-4.1-mini
   priced (openai public list 2026-07-14: 0.40/1.60 per 1M); explicit `pricingStatus`
   (priced | unpriced_model | tokens_unavailable); an unpriced model emits a loud
   structured cost-log warning with model id + request id while preserving all usage
   data; no guessed prices; telemetry only.
3. **Operator doctrine** (vault): `v1-concierge-operations-runbook-v1` (full daily
   workflow: opening checks, screening, generation, pregame delivery, settlement,
   postgame delivery, shutdown, R1-R14 bounded recovery with retry/auth/spend/record/
   stop per path); `v1-delivery-ledger-template-v1` (delivery ledger + operator-time
   log, PII-out-of-vault rules, freeze-doc-aligned $/operator-hour metric);
   `v1-rc-drill-package-v1` (prepared drill: caps 2 calls / 2 runs, four separate
   authorization gates, criterion-mapped script incl. a source-outage-class forced
   recovery, pass/fail + evidence + rollback + report format).

## verification

DevCore.Api.Tests **1235/1235** (+23); agent-service pytest **453/453** (+5); vitest
**134/134**; bundle compiles. Guard proven: zero service invocations and zero rows on a
409; simultaneous identical requests -> exactly one run + one 409. Review (2 angles):
12 findings fixed (orientation-insensitive identity, normalization tolerance,
collation-independent competition compare, bounded gate dictionary, deterministic 409
correlation id, runbook path/precondition/PII/metric/drill-mapping corrections),
2 refuted with evidence. RC criterion-1 wording confirmed already corrected. Runtime
never started (docs review verified every referenced route/script against source).

## deferred (not authorized, not started)

1. WI-0013 integration and push.
2. RC drill execution (four gates, all closed).
3. Multi-instance db uniqueness constraint (schema; needs its own WI).
4. Guard alias canonicalization beyond the screened-list workflow (documented limitation).

### Slice Synopsis

**Change:** Shipped the fail-closed duplicate-creation guard and full metering price
coverage, and wrote the executable V1 operator doctrine (runbook, ledgers, prepared RC
drill package).
**Reason:** Third critical-path item -- duplicate spend was unguarded, a configured model
metered as silent None, and the daily workflow lived in session history.
**Proof:** Red-first 1235/1235 C# + 453/453 pytest + 134/134 vitest; concurrency test
proves one-run-one-409; review 12 fixed / 2 refuted.
**State:** dai `85a8831` local on `wi/0013-pilot-ops-hardening`; vault docs committed
locally; nothing pushed; RC drill NOT executed; 0 paid calls, 0 captures, 0 DB writes;
runtime cold.
**Next:** Separately authorize WI-0013 integration and push, then the RC drill (own
gates).
