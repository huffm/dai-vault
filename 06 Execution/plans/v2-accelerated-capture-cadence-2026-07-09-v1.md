---
title: "V2 Accelerated Capture Cadence -- 2 slate days (2026-07-09, 2026-07-10)"
type: "plan"
date: "2026-07-08"
status: "AUTHORIZED -- operator directive 2026-07-09T01:40Z; executes in the 10:00-13:00 ET windows"
project: "DAI"
slice: "V2 Accelerated Capture Cadence v1"
related:
  - "04 Products/sports-v1/prompting/prompt-market-context-hardening-v1.md"
  - "06 Execution/patterns/capture-closeout-run-eligibility-rule-v1.md"
  - "06 Execution/reports/deliberate-divergence-settlement-and-n7-checkpoint-2026-07-08-v1.md"
  - "06 Execution/patterns/settlement-readiness-discipline-v1.md"
  - "06 Execution/reconciliations/canonical-reconciliation-residue-contract-v1.md"
---

# v2 accelerated capture cadence -- runbook

## authorization (operator, 2026-07-09T01:40Z)

- duration: next 2 MLB slate days = **2026-07-09 and 2026-07-10**.
- volume: up to **8 eligible runs per slate day**.
- goal: enough v2-era settled rows to move market-disagreement n=7 toward 10 AND test
  whether the hardening reduces attribution failures vs the frozen baseline
  (Pass 72 / FAIL 10 / Unclear 203).
- spend: <= **$0.05/day hard cap** on model calls (~8 x $0.0007 expected); each screen
  and each generation also burns a the-odds-api call. gpt-4o-mini only.

## execution window

10:00-13:00 ET (14:00-17:00Z). timing is load-bearing: evening slates leave pre-game by
late afternoon (2026-07-05 capture v1 failed on this). target ~10:20 ET start.

## per-day procedure

1. **pre-flight state:** docker/devcore-sql up; DevCore.Api :5007 up (Development);
   agent-service DOWN until capture (start DEFAULT config -- registry canary env NOT
   required: the hardened prompt is the LIVE prompt. optionally set
   DAI_MLB_REGISTRY_PROMPT_CANARY=1 process-scoped to land v2 ROUTE rows; preferred so
   rows carry full v2 provenance. .env NEVER edited).
2. **settlement pairing first (day 2+):** before any new capture, settle the PRIOR day's
   cohort if finals READY (check-settlement-finals.ps1 -> preflight-settlement.ps1 ->
   identity /reconcile with full residue). capture without settlement is not evidence.
3. **screen the full slate** via GET /api/agent-runs/source-readiness (one call per
   candidate; reuses generation retrieval).
4. **eligibility (ALL required, operator's criteria):** identityStatus=matched; starter
   level=enriched; market level=backed_depth with a clear consensus side; bookCount >= 5;
   the gamePk has NO existing active run (duplicate-active risk zero -- check /rows or
   reconcile-precheck first); game strictly pre-game.
5. **ranking among eligible (soft, take up to 8):** narrower de-vigged consensus gap
   first (closer games are likelier to produce dai-vs-market disagreement rows), then
   higher bookCount.
6. **generate** one run per selected game via POST /api/agent-runs
   (sports.matchup.analysis). after EACH run verify on /rows before the next:
   promptSource=registry with recipe ...backed_depth.v2@v2 (if canary env on) OR
   promptSource live with observedDataRegime=starter_enriched_market_backed_depth;
   attributionStatus=complete; attributionFidelityStatus != FAIL (see hard stops).
7. **closeout (capture-closeout-run-eligibility-rule-v1, binding):** capture-cadence
   runs ARE the intended prediction rows for their games -- they stay ACTIVE (do not
   exclude them; they are not evidence/QA runs). any extra diagnostic/retry run created
   along the way IS excluded diagnostic immediately. closeout evidence REQUIRED: per-run
   eligibility list + ZERO-row duplicate-active sweep + current-slice.md append +
   continuation handoff. agent-service restarted DEFAULT-OFF/stopped after capture.

## hard stops (operator's, verbatim intent -- any one halts the day)

1. duplicate-active gamePks > 0 at any check -> STOP (hygiene regression).
2. attribution guard FAILS unexpectedly on a new v2 run (FailMarketAttributionMismatch
   on a hardened run) -> STOP; that is the exact defect the hardening claims to prevent;
   capture more evidence of that run only, then halt for operator review.
3. registry falls back from v2 without explanation (fallbackReason not
   assembly_error-on-known-partial-shape) -> STOP.
4. a capture-closeout diagnostic exclusion is missed (sweep non-zero at closeout) ->
   STOP and fix before anything else.
5. daily spend exceeds $0.05 model cap -> STOP.
standing stops: any /rows row/eval inconsistency; odds/statsapi source unavailability.

## measurement rules

- v2-route rows are a DISTINCT regime era; never pool their attribution rates with v1.
- disagreement ledger reads remain unreadable until n >= 10; no edge language either way;
  candidate-edge wording only for CountsAsCandidateEdge (deliberate) rows.
- after both days settle: file a gate-4 evidence readout per cohort (template) + a
  Hardened-Regime Baseline Measurement v1 note comparing new-run guard outcomes to the
  frozen FAIL 10/285 baseline (rate direction only at these n).

## schedule as executed

session crons (session-only; if the session died, run this runbook manually per day):
- 2026-07-09 ~10:20 ET: day-1 capture (this doc, per-day procedure).
- 2026-07-10 ~10:20 ET: settle day-1 cohort, then day-2 capture.
- 2026-07-11 ~10:20 ET: settle day-2 cohort, gate-4 readout(s), baseline measurement
  note, cadence wrap report. capture authorization ENDS here; further capture needs a
  new operator go.
