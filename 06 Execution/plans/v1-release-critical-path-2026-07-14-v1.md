---
title: "V1 Release Critical Path v1 (frozen 2026-07-14)"
type: "plan"
date: "2026-07-14"
status: "active"
project: "DAI"
slice: "V1 Release Definition and Critical Path v1"
repos:
  dai: "unchanged (read-only audit at e64567f)"
  dai-vault: "docs-only"
tags:
  - planning
  - release
  - roadmap
related:
  - "04 Products/sports-v1/v1-release-definition-and-scope-freeze-v1.md"
  - "04 Products/sports-v1/buyer-validation-brief-v1.md"
  - "06 Execution/plans/platform-delivery-timeline-v1.md"
  - "06 Execution/reports/reconciliation-planning-readout-v1.md"
---

# v1 release critical path v1

Operator-authorized release plan (2026-07-14). Product definition and acceptance
criteria live in `v1-release-definition-and-scope-freeze-v1.md`; this document carries
the workflow audit, backlog triage, the three critical-path work items, and the dated
path. Implementation is NOT authorized by this plan -- each work item gets its own
authorization under the WI-0007 gate.

## release dates (operator-set 2026-07-14; audit found no infeasibility)

| milestone | date |
|---|---|
| V1 scope freeze | Fri 2026-07-17 (frozen early, 2026-07-14) |
| buyer workflow complete (WI-0011 + WI-0012 integrated) | Fri 2026-07-24 |
| release candidate (WI-0013 integrated + RC drill passed) | Fri 2026-07-31 |
| first paid private pilot | Fri 2026-08-07 |
| pilot evaluation | Fri 2026-08-21 |

## workflow audit (code evidence, dai e64567f, 2026-07-14)

| # | step | state | evidence |
|---|---|---|---|
| 1 | upcoming matchup discovery | **complete** | analyzer picker loads live schedule via GET /api/competitions/{code}/matchup-dates; operator screening via GET /source-readiness |
| 2 | matchup selection | **complete** | sport/level/team pickers + date pills (analyzer.component.ts:690-705); buyer-ready competitions filtered (NBA, MLB) |
| 3 | run creation | **complete, but unguarded** | POST /api/agent-runs from AnalyzerComponent; NO duplicate-active-run guard in UI or API (creation always inserts; AgentRunsController.cs:39-217) -> WI-0013 |
| 4 | result generation | **complete, operator-flagged** | live path default; hardened v2 registry route requires the process-scoped canary flag (operator-run generation in V1) |
| 5 | buyer-facing presentation | **partially complete** | analyzer renders the RAW POST response: numeric "75%" confidence tile (analyzer.component.html:328-331) + threshold "Strong/Mixed/Weak" labels (ts:162-165) = unsupported-claim surface; market context NOT rendered; gamePk never displayed; identity echoed from form state not the API; no-position rendering EXISTS and is distinct; buyer projection (/artifact/buyer) governs only the secondary panels -> WI-0011 |
| 6 | result retrieval / history | **missing (mocked)** | history page = MOCK_ENTRIES, no backend call; real GET /api/agent-runs/recent exists unused; account page static mock -> recap delivery covers V1; history page post-V1 |
| 7 | settlement | **complete, operator-only** | finals guard + strict preflight + identity /reconcile with full residue; idempotent; proven across 15 settlements |
| 8 | outcome display | **missing (buyer)** | GET /{id}/evaluation exists (auth-protected) but no buyer surface consumes it; buyer never sees a settled result -> WI-0012 |
| 9 | operator troubleshooting | **complete, operator-only** | /prompt-trace + dev/artifacts dashboard (double-gated: enableDevTools + enableDevBatchRuns, prod=false); WI-0004 shutdown script; WI-0005 cache fix |

Auth boundary (verified current): [Authorize(Policy=AgentRunAccess)] on the whole
controller; DevBypass requires IsDevelopment() AND Dev:EnableBypassAuth (fail-closed);
MSAL config-gated with fail-loud placeholder assertion. Payment/entitlement code: NONE
anywhere (verified broad search) -- V1 uses the manual Stripe link per the freeze doc.
Deployment: Dockerfiles + compose smoke exist, CI empty, prod frontend env points at
http://localhost:5007 -- hosted deploy is post-V1; the pilot runs on the local operator
stack by design.

## backlog triage

| item | class | reason |
|---|---|---|
| buyer confidence/evidence presentation (raw % tile, threshold labels, missing market context + identity) | **V1 blocker** | the buyer surface currently makes an unsupported implied-probability claim; contract conformance IS the product promise -> WI-0011 |
| buyer result delivery (rendered brief export) | **V1 blocker** | concierge delivery needs a deterministic buyer-safe rendering; today the only surface is the live app screen -> WI-0011 |
| settled outcome recap to buyer | **V1 blocker** | "settled outcome after completion" is in the product promise; no buyer surface shows it today -> WI-0012 |
| duplicate-active-run creation guard | **V1 blocker (small)** | release gate requires duplicate-free generation; today creation is unguarded at UI and API -> WI-0013 |
| operating runbook + failure recovery | **V1 blocker (docs)** | release gate requires runbook-only operation incl. recovery -> WI-0013 |
| release observability: cost-log pricing covers the configured model | **V1 blocker (small)** | usage gate; today metering prices gpt-4o-mini only while provisioning seeds gpt-4.1-mini -> None costs -> WI-0013 |
| access + payment | **V1 blocker (process, no code)** | manual Stripe payment link + delivery ledger per freeze doc; no implementation |
| daily matchup selection | **V1 covered (no new code)** | picker + source-readiness exist; runbook codifies |
| no-position behavior | **V1 covered (verify only)** | lean-null + pass/avoid postures already render distinctly; WI-0011 acceptance re-verifies |
| doubleheader capture operation (TB@BOS 07-17 packet) | **useful, nonblocking** | independent evidence activity under its own authorization; blocks release ONLY if it reveals a critical identity/persistence/reconciliation defect |
| secrets hygiene (rotate odds key + sa password sitting in local dev files) | **useful, nonblocking** | local-only, untracked; rotate before any access broadening; noted in runbook |
| real history page (wire GET /recent) | **post-V1** | recap delivery covers outcome traceability for one pilot buyer |
| hosted deploy (Entra prod values, CORS, prod apiBaseUrl, CI) | **post-V1** | concierge pilot needs none of it; deployment shape already exists |
| identity-status refinement | **post-V1 (dormant)** | replan trigger never observed |
| WI-0002 artifact chip alignment | **post-V1** | presentation polish; no buyer-value dependency |
| buyer copy polish (ledger entry 21: tone/cadence) | **post-V1** | label decisions land in WI-0011; broader tone pass follows real buyer feedback |
| /metrics denominator exclusion filtering | **post-V1** | changes tracked numbers; needs its own approval |
| EF global tenant filter | **post-V1** | single-tenant pilot; convention-scoping suffices at this scale |
| registry-authoritative default-ON | **post-V1** | locked layer; operator uses the process-scoped flag for pilot generation |
| WI-0003 shared chip module | **reject/defer indefinitely** | no second consumer exists (its own activation gate) |
| calibration volume expansion / model or prompt tuning | **reject for V1** | Gate 4 false; nothing is tuned from n=15 |

## critical path: three work items

### WI-0011 (proposed) -- Buyer Decision Brief Contract v1

- **problem:** the buyer result panel renders the raw POST response: a numeric
  confidence tile ("75%") and 0.70/0.45-threshold "Strong/Mixed/Weak" labels -- an
  implied-probability claim the evidence contradicts (discrimination inverted). Market
  context and persisted game identity (gamePk, generated-at) are absent. There is no
  deliverable rendering of the brief.
- **impact:** buyer -- the core deliverable becomes claim-safe, identity-stable, and
  email-deliverable.
- **scope:** (1) buyer result panel consumes buyer-safe fields only: numeric confidence
  and threshold labels removed from every buyer surface (internal/dev surfaces keep
  them); evidence-gated band becomes the only strength language. (2) identity block
  echoed from the persisted run (teams, gameDate, gamePk, generated-at). (3) minimal
  plain-language market-context line from persisted market fields (agreement, consensus
  side, book count). (4) deterministic buyer-brief export (markdown/HTML render of the
  buyer-safe read) usable for email delivery. (5) claim-safe checks extended to the
  export.
- **exclusions:** no prompt/model/scoring/confidence-VALUE change; no new signals or
  sources; no history/account pages; no delivery automation; no new schema.
- **acceptance:** no numeric probability or Strong/Weak label reachable on any buyer
  surface or export; identity on the export matches the persisted run and prompt-trace
  byte-for-byte; no-position runs render the explicit no-position form; export is
  deterministic for a fixed run; claim-safe copy tests green; existing suites green.
- **dependencies:** none. **sequence:** 1. **target:** complete + integrated by
  Wed 2026-07-22.
- **runtime behavior:** YES (presentation + a read-model/projection extension). Locked
  layers (prompt, model, confidence values, scoring, decision, reconciliation,
  calibration, Gate 4) untouched.
- **tests:** Angular component tests (tile removal, band-only strength, identity block,
  no-position), C# projection/contract tests (buyer fields incl. market context),
  export determinism test. **paid calls: 0** (existing persisted runs as fixtures).
  **DB writes: 0** (test databases only).

### WI-0012 (proposed) -- Settled Outcome Recap v1

- **problem:** a settled outcome never reaches the buyer; the product promise includes
  per-read outcome traceability.
- **impact:** buyer -- closes the loop on every delivered brief; operator -- recap is
  generated, not hand-written.
- **scope:** buyer-safe settled-recap projection (identity, what the read said, final
  score, per-read result) + deterministic recap export for email delivery; recap
  renders only for settled runs; excluded runs render an honest "no result -- event not
  evaluated" form.
- **exclusions:** no aggregate record/performance surfaces; no history page; no
  settlement-write changes; no schema change.
- **acceptance:** recap matches the persisted evaluation exactly; no aggregate claim
  anywhere; excluded-run form correct (823357 as fixture); export deterministic;
  suites green.
- **dependencies:** WI-0011 (export shape). **sequence:** 2. **target:** complete +
  integrated by Fri 2026-07-24 (buyer workflow complete).
- **runtime behavior:** YES (read-only surfaces). **tests:** projection + export tests
  incl. excluded-run fixture. **paid calls: 0. DB writes: 0.**

### WI-0013 (proposed) -- Pilot Operations Hardening v1

- **problem:** run creation has no duplicate-active-run guard (UI or API); cost
  metering prices only gpt-4o-mini so the configured default records None costs; no
  operator runbook exists for the daily pilot workflow or failure recovery.
- **impact:** operator + release risk -- duplicate-free generation, honest usage
  records, and a runbook-only operable workflow are all RC gates.
- **scope:** (1) creation-time duplicate-active-run guard (reject or require an
  explicit override when an active run exists for the same matchup identity/gamePk --
  design decided in-slice against the reconcile-precheck semantics; existing
  multi-run-per-gamePk settlement machinery unchanged). (2) metering pricing table
  covers the configured model(s); missing-price becomes a loud log warning, never a
  silent None. (3) operator runbook (vault doc): daily cadence screen -> generate
  (registry flag) -> render -> deliver -> settle -> recap; failure recovery (WI-0004
  shutdown, WI-0005 cache restart lesson, source-outage and model-error paths);
  delivery ledger + operator-minutes log; secrets-hygiene note. (4) RC drill: one full
  day executed from the runbook alone with one forced recovery, recorded as the RC
  record.
- **exclusions:** no scheduler/automation; no billing; no deploy; no /metrics change;
  no registry default change.
- **acceptance:** duplicate creation demonstrably guarded (tests + live check); cost
  log shows a real cost for the configured model; RC drill passes all 12 acceptance
  criteria from the freeze doc.
- **dependencies:** WI-0011/0012 integrated (the drill exercises them). **sequence:**
  3. **target:** complete + integrated by Wed 2026-07-29; RC drill by Fri 2026-07-31.
- **runtime behavior:** YES (small: creation guard + metering table). **tests:** guard
  matrix (C#), metering coverage test (python), drill checklist. **paid calls:** up to
  2 during the RC drill (the drill's generation, operator-authorized within the WI).
  **DB writes:** the drill's own run rows/outcome (normal product writes under the
  drill authorization); implementation tests: 0.

## dated critical path

| dates | work |
|---|---|
| Tue 07-14 | scope frozen (this plan + freeze doc committed) |
| Wed 07-15 .. Fri 07-17 | authorize + implement WI-0011 (scope-freeze date met by the standing freeze) |
| Fri 07-17 | independent: TB@BOS doubleheader packet reassessment (own authorization; NOT on this path) |
| Mon 07-20 .. Wed 07-22 | WI-0011 integration; authorize + implement WI-0012 |
| Fri 07-24 | WI-0012 integrated -- buyer workflow complete (milestone met) |
| Mon 07-27 .. Wed 07-29 | authorize + implement + integrate WI-0013 |
| Thu 07-30 .. Fri 07-31 | RC drill from runbook; RC record committed (milestone met) |
| Mon 08-03 .. Fri 08-07 | outreach per buyer-validation-brief (free sample -> Stripe link); first paid delivery day by Fri 08-07 |
| Fri 08-07 .. Fri 08-21 | pilot operation; ledger + metrics collection; evaluation 08-21 |

Slack: each WI has 1-2 buffer days; the path holds even if WI-0011 slips to 07-24,
because WI-0012 is small and WI-0013's drill is the only hard RC dependency.

## doubleheader experiment treatment

The 2026-07-17 TB@BOS doubleheader operation remains an independent evidence activity
under its own decision packet and authorization. It is NOT a release dependency and the
release timeline does not wait for it. It becomes release-relevant ONLY if it reveals a
critical identity, persistence, or reconciliation defect -- in which case that defect is
triaged against the RC gate on its own evidence.

## governance

This plan mints no WI (release-planning documentation). WI-0011/0012/0013 are PROPOSED
titles; each is created and authorized separately under the WI-0007 qualification gate
at implementation time. Authorization posture (no-spend; paid calls, captures,
reconciliation writes all not-authorized) is UNCHANGED by this plan; the WI-0013 RC
drill's 2 paid calls are authorized within that WI when it is opened, not now.
