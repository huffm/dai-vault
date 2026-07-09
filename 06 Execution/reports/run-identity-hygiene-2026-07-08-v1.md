---
title: "Run Identity Hygiene v1: 824662 + 823281 duplicate-active audit (read-only)"
type: "evidence-report"
date: "2026-07-08"
status: "COMPLETE -- audit + operator-approved exclusions applied; both gamePks single-active"
project: "DAI"
slice: "Run Identity Hygiene v1"
repos:
  dai: "unchanged (a0db824)"
  dai-vault: "docs-only"
related:
  - "04 Products/sports-v1/run-eligibility-and-supersession-contract-v1.md"
  - "06 Execution/reports/backed-depth-capture-settlement-handoff-2026-07-08-v1.md"
  - "06 Execution/reports/market-attribution-fidelity-guard-handoff-2026-07-07-v1.md"
  - "06 Execution/reconciliations/outcome-reconciliation-readiness-next-slice-2026-06-30.md"
---

# run identity hygiene v1 -- duplicate-active audit for 824662 + 823281

## 1. objective

resolve or explicitly classify the duplicate-active run condition on gamePks 824662 and
823281 so future settlement and calibration rows are identity-safe, without changing
model behavior, prompts, buyer copy, confidence logic, thresholds, reconciliation
semantics, or capture cadence. read-only diagnosis first; writes only per the 07-08
handoff section 13, which gates POST /{id}/exclude behind an in-session operator
approval of the proposal. this report is the diagnosis + proposal; NO exclusion has
been applied.

## 2. starting state

- dai: main @ a0db824, synced 0/0 with origin; only the known phantom
  DevCore.Data.csproj line-ending delta (untouched).
- dai-vault: main @ ccf4b3a, synced 0/0; two known intentionally-untracked files
  (preflight manifest json, system-state-synopsis) left untracked.
- devcore-sql docker container: UP (found up, ~2h). DevCore.Api :5007: found DOWN,
  started this slice via dotnet run (http profile), left RUNNING.
- agent-service (:8000): never started. paid model calls: 0. db writes: 0.
- 07-07 cohort settled 6/6 earlier today; capture cadence PAUSED; gate 4 FALSE
  (discrimination_inverted, insufficient_market_disagreement).

## 3. files / endpoints / scripts inspected

- GET /api/agent-runs/prompt-route-calibration/rows (live, read-only; 285 rows;
  5 rows matched the two gamePks).
- devcore db via container sqlcmd, read-only SELECTs: AgentRuns eligibility fields
  (ExclusionReason, SupersededByAgentRunId), AgentRunOutcomes / AgentRunEvaluations
  counts, duplicate-active gamePk sweep.
- 04 Products/sports-v1/run-eligibility-and-supersession-contract-v1.md (authority
  for exclusion reasons, matcher behavior, per-run asymmetry).
- 06 Execution/reports/backed-depth-capture-settlement-handoff-2026-07-08-v1.md
  section 12/13 (authorization + approval gate).
- origin slices in current-slice.md: directional contrast cohort v4 (06-28 morning),
  real-cohort live soak v1 (06-28 evening, 14 runs), registry canary real
  confirmation v1 (1 run), starter-missing regime capture v1 (4 runs), live batch +
  settlement reconciliation gate (06-29/06-30, runs 260016-260023).
- controllers: AgentRunsController.cs (rows endpoint tenant scoping, read-only).

## 4. duplicate pair findings

### 4a. the flagged pairs are actually a triple and a pair (5 rows, not 4)

live /rows + db agree exactly. all five rows tenantKey 1, provider mlb_statsapi,
Status completed, ExclusionReason NULL (active), SupersededByAgentRunId NULL.

```text
gamePk 823281 (LAD @ SD, game 2026-06-28 20:10Z) -- THREE active, all unsettled (0 outcomes / 0 evals):
  240026  6a37433e-f36b-1410-816c-00373db4b724  started 06-28 13:44Z (pregame)
          origin: directional contrast cohort v4 (morning calibration cohort)
          lean home 0.75; market away, 9 books, medians 0.459/0.582; marketAgreement=false
          guard: Pass / prose_acknowledges_market_opposition / DeliberateDivergence
  250027  1ede423e-f36b-1410-816d-00373db4b724  started 06-28 22:55Z (IN-GAME, ~2h45m after first pitch)
          origin: real-cohort live soak v1 (capture/soak evidence batch, 14 runs over the day's slate)
          lean home 0.70; market away, 6 books, DEGENERATE medians 0.091/0.971 (in-game odds snapshot)
          guard: Pass / prose_acknowledges_market_opposition / DeliberateDivergence
  250028  21de423e-f36b-1410-816d-00373db4b724  started 06-29 00:42Z (post/late-game)
          origin: registry canary real confirmation v1 (the single approved canary run)
          lean home 0.675; NO market context (regime starter_enriched_market_missing)
          guard: UnclearMarketAttribution / no_market_consensus / UnclearDivergence

gamePk 824662 (SD @ CHC, game 2026-06-30 00:05Z) -- TWO active, one settled:
  250030  2cde423e-f36b-1410-816d-00373db4b724  started 06-29 01:28Z (pregame, day before)
          origin: starter-missing regime capture v1 (away starter TBD; regime-capture evidence run)
          lean home 0.675; market home, 2 books; marketAgreement=true; UNSETTLED
          guard: UnclearMarketAttribution / both_market_directions_asserted / UnclearDivergence
  260023  4cbd433e-f36b-1410-816e-00373db4b724  started 06-30 01:10Z (pregame, registry-routed)
          origin: live batch 06-29 (registry-authoritative, backed_depth, full provenance)
          lean home 0.70; SETTLED home_win 06-30 via EXPLICIT per-run path (1 outcome + 1 eval);
          settlementNotes record the collision: "identity 824662 collides with prior cohort run
          250030 (MultipleMatches); explicit per-run outcome for live batch run 260023"
          guard: FailMarketAttributionMismatch / AccidentalDivergence
```

### 4b. classification of the issue

data hygiene, not selection-logic defect and not missing read-model visibility:

- the matcher works as the supersession contract specifies: two active runs sharing
  (SourceProvider, ExternalGameId) -> MultipleMatches, writes nothing, never guesses.
  demonstrated live on 824662 during the 06-30 settlement (readiness doc records it,
  and 260023's settlementNotes carry the evidence).
- the duplicates exist because evidence/capture/canary batches (soak, regime capture,
  canary confirmation) executed real games that calibration cohorts had already run
  or would run, and automatic-supersession-at-generation is explicitly deferred in
  the contract -- exclusion is a deliberate operator action that was never performed.
  the 06-30 readiness doc named 824662's duplicate and deferred it out of scope.
- two of the three 823281 duplicates are not even pregame rows: 1ede423e ran in-game
  (degenerate 0.091/0.971 market snapshot) and 21de423e ran post/late-game with no
  market. neither is a valid prediction row for calibration purposes.

## 5. remediation performed or deferred

performed: NONE. zero writes of any kind (verified: this slice issued only GETs and
read-only SELECTs; outcomes/evals counts unchanged; AgentRuns unchanged).

proposed (requires operator approval per handoff sec 13 before any POST /{id}/exclude):

```text
1. exclude 1ede423e-f36b-1410-816d-00373db4b724  reason=diagnostic
   (in-game soak/capture evidence run; degenerate market snapshot; not a pregame prediction)
2. exclude 21de423e-f36b-1410-816d-00373db4b724  reason=diagnostic
   (canary plumbing confirmation run; post/late-game; no market context)
3. exclude 2cde423e-f36b-1410-816d-00373db4b724  reason=superseded,
   supersededByAgentRunId=4cbd433e-f36b-1410-816e-00373db4b724
   (regime-capture run replaced by the settled registry-routed run; provenance link
   preserved -- exactly the remediation the 06-30 readiness doc anticipated)
4. keep 6a37433e-f36b-1410-816c-00373db4b724 ACTIVE
   (authoritative pregame v4 calibration run for 823281)
5. keep 4cbd433e-f36b-1410-816e-00373db4b724 ACTIVE (settled, authoritative for 824662)
```

EXPLICIT FLAG (sec-13 stop condition): the exclusions themselves create no ledger
entries. but they make 823281 single-active on a guard-classified DeliberateDivergence
row (6a37433e). if a LATER slice settles 823281, that settlement would create the
FIRST deliberate CountsAsCandidateEdge ledger entry AND move marketDisagreementN
6 -> 7, firing the n=7 re-projection checkpoint. settling 823281 is NOT part of this
proposal and needs its own approval.

## 6. validation evidence

- duplicate-active condition explicitly classified for both gamePks: 823281 = triple
  (1 authoritative pregame + 2 non-pregame evidence runs); 824662 = pair (1 settled
  authoritative + 1 anticipated-supersession capture run). section 4 tables are
  reproducible via the queries in section 3.
- pre-settlement selection cannot silently pick a wrong run TODAY: the identity
  matcher returns MultipleMatches and writes nothing on ambiguity (contract section
  "matcher behavior"; live demonstration 06-30 on 824662). after the proposed
  exclusions, each gamePk has exactly one active run -> SingleMatch semantics.
- no already-settled row is re-reconciled: no /reconcile or /{id}/outcome call was
  made; outcomes/evals remain 1/1 on 260023 and 0/0 on the other four.
- no outcome/evaluation double-write occurred: db counts verified before/after audit
  (no writes issued).
- no unrelated runs mutated: no mutation issued at all.

## 7. remaining risks

- the duplicate-active surface is FAR wider than the two flagged gamePks: a read-only
  sweep found 19 gamePks with >1 active run (823281 and 824744 have three). most stem
  from the 06-28 real-cohort live soak duplicating that day's slate and the capture
  batches. left untouched per the no-broad-cleanup constraint.
- sharpest of these: gamePk 823613 (CHC @ NYM) has TWO ACTIVE SETTLED runs with
  OPPOSITE leans (200018 beb5433e away 06-22; 220014 d879433e home 06-24), each with
  1 outcome + 1 eval -- one real game contributing two contradictory rows to the
  valid calibration denominator (run-weighted today). needs its own scoped decision
  (possibly a postponed/rescheduled game analyzed twice).
- 824818 and 825066 each have 1 settled + 1 active-unsettled run (same shape as
  824662) and will MultipleMatch if anyone identity-reconciles them.
- until the operator approves the proposal, 823281 remains triple-active and cannot
  be settled by the identity path; 824662 remains MultipleMatches-on-identity (its
  settled row is safe and unaffected).
- exclusion reasons are operator judgment calls: diagnostic-vs-superseded for
  2cde423e is defensible either way; superseded chosen for the provenance link.

## 8. next recommended action

1. operator approves / amends the section-5 proposal; apply the three POST
   /{id}/exclude calls; re-read /rows + the duplicate sweep to confirm both gamePks
   single-active; append the result to this report.
2. separately decide whether to settle 823281 on 6a37433e (would create the first
   deliberate ledger entry + fire the n=7 checkpoint -- own approval).
3. scope Run Identity Hygiene v2 for the remaining 17 duplicate-active gamePks,
   823613's double-settled contradiction first.
4. then Prompt Market Context Hardening v1 (approval-gated), per the standing
   sequence.

## 9. remediation applied (post-approval addendum, 2026-07-08)

operator approved the section-5 proposal in-session ("Approve all 3") the same day.
applied via POST /api/agent-runs/{id}/exclude (three calls, each echoed the stored
state):

```text
1ede423e-f36b-1410-816d-00373db4b724 -> exclusionReason=diagnostic
21de423e-f36b-1410-816d-00373db4b724 -> exclusionReason=diagnostic
2cde423e-f36b-1410-816d-00373db4b724 -> exclusionReason=superseded,
    supersededByAgentRunId=4cbd433e-f36b-1410-816e-00373db4b724
```

post-write verification (fresh db reads):

- 823281 active runs: 1 (6a37433e only). 824662 active runs: 1 (4cbd433e only).
  both gamePks now SingleMatch-safe on the identity path.
- AgentRunOutcomes 124 / AgentRunEvaluations 124 -- UNCHANGED (exclusion is a soft
  eligibility flag; no settlement rows written, no settled row touched, no
  re-reconciliation).
- total excluded runs 16 -> 19 (+3, exactly the approved set); runs/artifacts/traces
  preserved (soft flag, never a delete).
- valid calibration denominator unaffected: none of the three excluded runs was
  settled, so valid-settled stays 122.

the section-5 sec-13 flag stands: 823281 is now single-active on a guard-classified
DeliberateDivergence row; settling it is a separate, approval-gated decision.
