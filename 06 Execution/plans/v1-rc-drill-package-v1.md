---
title: "V1 Release-Candidate Drill Package v1 (prepared, NOT executed)"
type: "plan"
date: "2026-07-14"
status: "prepared"
project: "DAI"
slice: "WI-0013 Pilot Operations Hardening v1"
repos:
  dai: "referenced (no change by this doc)"
  dai-vault: "docs-only"
tags:
  - release
  - drill
  - operations
related:
  - "04 Products/sports-v1/v1-release-definition-and-scope-freeze-v1.md"
  - "06 Execution/plans/v1-concierge-operations-runbook-v1.md"
  - "06 Execution/plans/v1-delivery-ledger-template-v1.md"
---

# v1 release-candidate drill package v1

**Prepared under WI-0013. NOT executed. Every paid or write step below requires its own
explicit operator authorization at drill time.** Target drill date: on or before Friday
2026-07-31.

## authorization boundaries (four distinct gates -- never merged)

1. **Pregame drill authorization** (drill day): covers screening + up to the capped
   generation + brief rendering. Grants the paid caps below and NOTHING else.
2. **Settlement authorization** (day after finals): covers the reconciliation writes for
   the drill runs only.
3. **Payment-link test authorization**: covers the Stripe TEST-MODE dry-run only
   (explicitly marked test transaction; no real charge).
4. **Real buyer delivery is NOT part of the RC drill.** The drill's "delivery" step sends
   to the operator's own address.

## environment and commits

- Local operator host per the runbook opening checks (section 1).
- dai == origin/main at the WI-0013-integrated commit (recorded at integration; verify
  exact hash at drill time). dai-vault == origin/main current.
- Registry v2 route process-scoped at generation, `.env` untouched.

## candidates

One or two MLB games selected AT DRILL TIME from that day's slate via the runbook
screening procedure (section 2): eligible source-readiness, stable gamePk identity, no
active duplicate. Named in the drill-day authorization before any spend.

## hard caps

- paid model calls: **max 2**
- created runs: **max 2**
- external-source calls: screening + generation odds-api calls for the named candidates
  only (max 2 screens per candidate incl. one R9 re-screen)
- product-row writes: the drill's own run rows + (under gate 2) their outcomes/evaluations
- reconciliation writes: only under gate 2, only the drill runs, full residue
- everything else: zero writes, zero prompts/models/config changes

## drill script (maps to the freeze-doc RC criteria 1-12)

1. Opening checks from the runbook alone (criterion 11 partial) -- cold start, approved
   commits, secrets presence, health.
2. Payment-link dry-run per runbook section 7, test mode, ledger `entitlement=test`
   (criterion 1).
3. Screen the slate; select + record candidates (criterion 2).
4. Generate run 1 (canary): verify no duplicate (criterion 3 -- ALSO deliberately re-POST
   the same game once and confirm the 409 with zero spend), registry v2 provenance,
   identity == requested gamePk (criterion 6), priced metering line (criterion 8).
5. Render /brief + /brief/markdown; check contract conformance + no numeric confidence
   (criteria 4, 5); "deliver" to the operator address; complete the ledger row.
6. **Forced recovery scenario (criterion 7 -- source outage class, per the freeze doc):**
   before generating run 2 (or after run 1 if only one candidate), run one readiness
   screen with a deliberately INVALID odds-api key set process-scoped in the shell --
   the screen observably fails as a source outage; recover per runbook R4/R9: restore
   the correct environment, restart the affected process, re-screen once, and verify the
   day's state is intact (no duplicate row, no lost run). The extra screen counts inside
   the external-call cap. Optionally ALSO demonstrate the R10 API stop/restart path
   (WI-0004 script) -- a bonus, not the criterion-7 evidence.
7. If a second candidate was authorized: generate run 2 within caps.
8. Shutdown to cold via the runbook (criterion 11).
9. Next day under gate 2: finals guard -> strict preflight -> feed/live re-verify ->
   identity /reconcile with full residue -> verify outcome attaches to the correct run
   (criterion 9); idempotent re-write attempt 409s.
10. Render /recap + /recap/markdown; verify state + score + per-read result (criterion
    10); "deliver" to the operator address; complete the ledger row.
11. Write the RC record (criterion 12) in the post-drill report format below.

## pass/fail

PASS = all 12 freeze-doc criteria demonstrably satisfied with evidence captured; caps
respected; zero unauthorized writes; runtime returned to cold both days. Any criterion
unmet or any cap breached = FAIL: stop, record, remediate under a fresh decision. A FAIL
does not consume the 07-31 date automatically -- one re-drill is schedulable within the
window if remediation is docs/ops-only; engineering remediation mints a WI first.

## evidence to capture

Per criterion: command + output snippet or artifact reference (run ids, 409 body, trace
fields, metering line, markdown hashes -- fetch each markdown twice and compare, ledger
rows, preflight exit codes, recovery timeline, port checks). Store in the RC record.

## rollback / abort

Abort at any point: stop generation, shut down via runbook section 8, record state. Runs
already created remain (they are honest product rows); settle or exclude them under gate 2
rules. No history rewrites, no row deletion.

## post-drill report format (the RC record)

`06 Execution/reports/v1-rc-drill-record-<date>-v1.md` with: environment + commits;
authorizations cited; per-criterion PASS/FAIL + evidence; caps used (calls/runs/cost);
recovery scenario timeline; deviations; verdict (RC met / not met); Slice Synopsis.
