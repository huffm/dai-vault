---
title: "823281 Deliberate-Divergence Settlement + n=7 Disagreement Checkpoint Re-Projection"
type: "evidence-report"
date: "2026-07-08"
status: "COMPLETE -- first deliberate ledger entry settled (incorrect); n=7 checkpoint re-projected"
project: "DAI"
slice: "Settle 823281 + n=7 Checkpoint v1"
repos:
  dai: "unchanged (a0db824)"
  dai-vault: "docs-only"
related:
  - "06 Execution/reports/run-identity-hygiene-v2-2026-07-08-v1.md"
  - "06 Execution/reports/evidence-acquisition-cadence-proposal-2026-07-07-v1.md"
  - "06 Execution/reports/gate4-evidence-sufficiency-projection-2026-07-07-v1.md"
  - "06 Execution/patterns/market-attribution-fidelity-guard-v1.md"
---

# 823281 settlement + n=7 checkpoint re-projection

taxonomy language rules apply: persisted marketAgreement=false rows are "market-opposed
rows"; "candidate edge signal" is reserved for deliberate divergences.

## 1. settlement readout (single run, operator-ordered)

- gate discipline: check-settlement-finals.ps1 READY 1/1 (Final/F, LAD 4 - SD 2) ->
  preflight-settlement.ps1 exit 0 (SingleMatch, 0 blockers; 3 provenance warnings
  expected on a pre-registry legacy v4 run) -> identity POST /reconcile, SingleMatch,
  full residue (statsapi_final / "gamePk 823281" / hygiene-follow-up note).
- run 6a37433e (240026, directional-contrast cohort v4, LAD@SD 06-28, lean home
  san-diego-padres 0.75, market consensus away 9 books): settled **away_win ->
  INCORRECT**. market side correct.
- live guard fields at settlement: attributionFidelityStatus=Pass /
  attributionFidelityReason=prose_acknowledges_market_opposition /
  divergenceInterpretation=**DeliberateDivergence** -> this row DOES count as a
  candidate edge signal (CountsAsCandidateEdge). it is the FIRST entry in the
  deliberate ledger, and it is a MISS: **deliberate ledger = 0 correct / 1 incorrect.**
- single-active precondition held (hygiene v1): exactly one active run for 823281;
  no MultipleMatches, no double-write. db: outcomes/evals 124 -> 125 each;
  valid-settled 121 -> 122; capture-closeout sweep still 0 duplicate-active gamePks.

## 2. n=7 checkpoint -- binding numbers (discrimination_hybrid_v1, live post-write)

- conclusionsAllowed = false; failingReasons = [discrimination_inverted,
  insufficient_market_disagreement].
- marketDisagreementN = 7 (checkpoint FIRED per cadence proposal section 8.7);
  readable=false (needs 10). market-opposed ledger: **2 correct / 5 incorrect
  (0.2857)** vs market-agree 36/58 (0.6207). deliberate subset 0/1; accidental 2/6.
- discrimination: still INVERTED, delta -0.1184 (gte_0.80 n=17 acc 0.4706 below
  0.75_0.79 n=73 acc 0.5890). note the delta moved -0.1321 -> -0.1184 purely from
  hygiene v2 (removing 823613's lucky-correct lowered the bottom band) plus this 0.75
  miss -- no discrimination progress, arithmetic drift only.
- marketCoverage 0.625 (>= 0.60, MET). validDirectional 104, reconciled 122,
  excludedRowCount 38, slates 14.

## 3. re-projection (what n=10 costs and what it can/cannot buy)

- gap: +3 settled market-opposed rows. observed divergence yield holds at ~1 per
  6-run capture cohort (07-06: 1/6, 07-07: 1/6) -> ~3 cohorts ~= 18 captured runs
  ~= 3 qualified capture mornings ~= 1.5-2 weeks at the approved cadence
  (2 mornings/wk, <=6 runs, $0.10/wk bound). model cost ~= $0.013 at the ~$0.00071/run
  anchor (each run also burns a the-odds-api call).
- free-row sources are EXHAUSTED: the legacy deliberate corpus is fully resolved
  (823284, 823608 excluded invalid; 823281 settled; its soak twin excluded
  diagnostic). every further disagreement row must be captured and paid for.
- what n=10 buys: satisfies insufficient_market_disagreement (the ledger becomes
  readable). what it CANNOT buy: discrimination_inverted -- outcome-dependent, not
  volume-purchasable (standing projection warning, re-confirmed). even at n=10,
  conclusionsAllowed stays false unless the inversion resolves on its own evidence.
- edge-narrative status at n=7: NEGATIVE. dai is 2/7 when opposing the market and the
  single deliberate divergence lost. no buyer-facing claim is licensed in any
  direction (small n; taxonomy language rules stand).

## 4. what this readout does not license

no tuning, no threshold edits, no model replacement, no buyer accuracy/edge claims,
no gate edits, no capture resumption (separate operator re-approval), no registry
default change.

## 5. decision queued for the operator

Prompt Market Context Hardening v1 (approval-gated, baseline Pass 72 / FAIL 10 /
Unclear 203 on 285 rows): its hygiene precondition is MET and the checkpoint read is
done. it does not attack either failing reason directly, but it protects the ONLY
instrument this program is about to spend money on -- captured divergence rows are
readable evidence only if prose attribution is trustworthy at generation time.
recommended sequencing: hardening FIRST (with its paid canary), THEN capture-cadence
resumption toward n=10.
