---
title: "July 22 Core Canary Settlement Reconciliation 2026-07-23 v1"
type: "reconciliation"
date: "2026-07-23"
status: "complete -- one eligible run settled correct via canonical identity reconcile; cumulative gates unchanged"
project: "DAI"
slice: "daily evidence operation 2026-07-23 (reconciliation phase; no work item)"
repos:
  dai: "unchanged (read-only; integrated CLIs and API invoked from main 85af96d)"
  dai-vault: "docs only; local branch ops/2026-07-23-daily-evidence, NOT pushed"
tags:
  - evidence-operations
  - sports-v1
  - reconciliation
  - settlement
related:
  - "06 Execution/patterns/settlement-readiness-discipline-v1.md"
  - "06 Execution/patterns/daily-evidence-acquisition-operating-workflow-v1.md"
  - "06 Execution/reports/exact-identity-core-canary-capture-2026-07-22-v1.md"
  - "06 Execution/reconciliations/canonical-reconciliation-residue-contract-v1.md"
---

# july 22 core canary settlement reconciliation 2026-07-23 v1

## purpose

Settle the single eligible July 22 capture -- the exact-identity core canary run -- against
the authoritative StatsAPI final, under a separate explicit operator authorization dated
2026-07-23. The July 22 capture record explicitly deferred settlement to "its own separate
authorization"; this is that authorization executed.

## inventory of july 22 artifacts

Exactly one run required reconciliation. All other July 22 artifacts were free-gate
observations or unrelated operations and were excluded:

| artifact | disposition |
|---|---|
| `<DAI_WORKSPACE_ROOT>/daily-evidence-capture-2026-07-22/exact-core-canary-1/` | THE eligible run (settled below) |
| `<DAI_WORKSPACE_ROOT>/events-gate-2026-07-22/attempt-1,2/` | free zero-quota observations, NO-GO 1 < 4; nothing to settle |
| `<DAI_WORKSPACE_ROOT>/daily-evidence-flight-2026-07-22/attempt-1/` | free pass-1 + preflight; nothing to settle |
| `<DAI_WORKSPACE_ROOT>/screen-2026-07-22/` | WI-0035 Slice 3 live screen executed 2026-07-20; unrelated operational test, 0 credits charged |

## eligibility gate

- identity: agent run `a9b0433e-f36b-1410-8191-00373db4b724`, tenant 1, provider
  `mlb_statsapi`, external game id `823438` (Los Angeles Dodgers @ Philadelphia
  Phillies), frozen flight `flight-2026-07-22-exact-core-canary-1`, freeze fingerprint
  `0d44530e2985ab0d4d6d460263306ef113f32171a73c1ecc5803a3151bf8f954`, provider event
  `111a955795876d50988b15c219ce0796` (exact match, delta 0s), lane `core`, no
  doubleheader ambiguity (single game, `double_header=N`).
- finality: `check-settlement-finals.ps1 -Competition mlb -GamePks 823438` returned
  READY exit 0 -- `abstractGameState=Final`, `codedGameState=F`, final (9-5), 9 innings.
- market evidence: frozen paid-screen bundle sha-256
  `d45b974eef6779e397afe6efac6a15c6133cce15bd5a60dafad623bfb63de379` (h2h+spreads, 9
  books, market consensus side = away).
- not previously settled: reconcile-precheck returned `SingleMatch`, `hasOutcome=false`,
  1 active unreconciled run.

## settlement execution (canonical path)

Order followed per settlement-readiness-discipline-v1: finals guard, then read-only
`preflight-settlement.ps1` (exit 0, ready 1, blockers 0, warnings 5), then the identity
reconcile POST.

- write path: `POST /api/agent-runs/reconcile` (single canonical writer; writes only on
  `SingleMatch`).
- outcome written: `away_win`, home 5, away 9, source `manual`, sourceRef
  `runId a9b0433e-f36b-1410-8191-00373db4b724`, notes carrying the pass label, statsapi
  final, flight id, provider event id, binding fingerprint, and the canary caveat
  (complete residue per the canonical reconciliation residue contract v1).
- response: `matchKind=SingleMatch`, `evalStatus=correct`.
- evaluation row `2ce9423e-f36b-1410-8192-00373db4b724`: lean `away`, winning side
  `away`, evaluated 2026-07-23T15:04:21Z.
- direction integrity: prose lean ("slight lean toward Los Angeles Dodgers") and encoded
  lean (`away`) agree; no contradiction refusal.
- flight-selection strata writeback verified on the read model: `evidenceStratum=core`,
  flight id and freeze fingerprint intact, `realizedVia=scheduled`, positions 1/1.

## grading proof

| field | frozen value | outcome |
|---|---|---|
| prediction | lean away (Dodgers), confidence 0.75, posture monitor | -- |
| market position | consensus away (median implied away 0.545 / home 0.493, 9 books) | market-agreed |
| authoritative final | LAD 9 @ PHI 5, Final/F, 9 innings (statsapi) | away win |
| classification | -- | **correct** |
| stratum | core (single-run technical canary, NOT measurement-grade cohort) | -- |
| exclusion | none (`exclusionReason` null) | counts in valid settled set |

The five preflight warnings (live prompt source, legacy fallback used, fallback reason
`disabled`, attribution partial, empty selected regime) are inherent to the canary's
routing posture, were recorded, and did not escalate; no `-Require*` strictness applied.

## cumulative evidence refresh (before -> after)

Read models re-run after settlement (pooled calibration readout, calibration metrics):

| metric | before | after |
|---|---|---|
| reconciled rows | 137 | 138 |
| unreconciled rows | 145 | 144 |
| matched (correct) rows | 80 | 81 |
| match rate (observed) | 0.5839 | 0.5870 |
| valid settled set (settled AND exclusion null) | 59 | 60 |
| settled slates pooled | 17 | 18 |
| directional N (pooled readout) | 134 | 135 |
| market-agreement sample | 85 (51c/34i) | 86 (52c/34i) |
| market-disagreement sample | 9 | 9 (unchanged; this run was market-agreed) |
| discrimination | inverted (delta -0.062) | inverted (unchanged) |
| gate 4 / conclusions_allowed | FALSE | FALSE (unchanged) |

One settled core canary is directional evidence only; the sample remains insufficient
for any calibration claim. The discrimination-hybrid verdict is unchanged: failing
reasons `discrimination_inverted` and `insufficient_market_disagreement` both stand
(market-disagreement readable N = 9 < 10).

## boundaries honored

No paid odds call, no capture, no source-code or schema change, no merge, no push, no
remote branch, no PR. Settlement wrote exactly one outcome row and one evaluation row
through the canonical API path. The dirty WI-0035 vault worktree and its six preserved
paths were not touched.
