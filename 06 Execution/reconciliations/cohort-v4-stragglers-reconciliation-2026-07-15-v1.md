---
title: "Cohort V4 and Historical Stragglers Reconciliation v1 (2026-07-15)"
type: "reconciliation"
date: "2026-07-15"
status: "complete"
project: "DAI"
slice: "Cohort V4 and Historical Stragglers Reconciliation v1"
repos:
  dai: "unchanged (runtime writes via existing endpoints only, at 85a8831)"
  dai-vault: "docs-only (this record)"
tags:
  - reconciliation
  - settlement
  - calibration
related:
  - "06 Execution/reports/preflight-settlement-manifest-2026-07-06-v1.json"
  - "04 Products/sports-v1/calibration/directional-contrast-cohort-v4-market-baseline-v3.md"
---

# cohort v4 and historical stragglers reconciliation v1

Operator-authorized settlement (2026-07-15) of the 18 identity-safe, officially-Final,
unsettled runs discovered by the Midweek Settlement Sweep: cohort v4 (2026-06-28 slate,
13 morning captures + the documented Phillies@Mets clean replacement 02de423e +
06-29 Marlins@Rockies) plus stragglers 824993 (06-15), 823932 + 823526 (07-04 v8
partial). Authorization waived ONLY the enumerated pre-registry provenance warnings
(promptSource != registry, attribution incomplete, regime != v2 target, 3
non-directional leans, 3 missing staged market baselines); blockers were 0 everywhere.

## write discipline (all satisfied)

- every score re-verified from statsapi feed/live IMMEDIATELY before its write
- precheck re-verified per game: IdentitySafe, exactly 1 active run, 0 outcomes
- identity POST /api/agent-runs/reconcile; SingleMatch on all 18; matched run ids ==
  the sweep inventory exactly
- full residue on every write (source=mlb_statsapi, sourceRef=gamePk, notes naming
  this authorization) per ADR 0006
- idempotency verified after settlement: re-POST of all 18 -> 18x HTTP 409, 0 writes
- write count exact: outcomes 140 -> 158 (+18), evaluations 158, 0 orphans
- untouched: 824766/824737 (RC drill identities, 0 runs), stale pending 087a433e,
  the 102 identity-less legacy rows, all 40 exclusions
- 0 model calls, 0 paid-source calls, 0 captures, 0 code/prompt/schema changes
- runtime returned to cold (stop script exit 0; ports free; docker down)

## results (18/18 settled; exactly as predicted by the sweep packet)

| pk | run | outcome | eval |
|---|---|---|---|
| 824256 | 3b37433e | away_win 7-5 | correct |
| 824821 | 3c37433e | away_win 6-4 | incorrect |
| 823362 | 4337433e | home_win 9-4 | correct |
| 822795 | 4537433e | away_win 3-2 | incorrect |
| 824422 | 4b37433e | home_win 6-5 | correct |
| 822959 | 4c37433e | home_win 5-1 | correct |
| 823686 | 5237433e | home_win 3-2 | correct |
| 824580 | 5437433e | away_win 5-4 | incorrect |
| 823769 | 5737433e | away_win 4-3 | incorrect |
| 823037 | 5d37433e | home_win 2-1 | correct |
| 824011 | 6237433e | home_win 4-1 | incorrect |
| 823204 | 6437433e | home_win 3-2 | incorrect |
| 824744 | 7137433e | home_win 5-4 | correct |
| 823608 | 02de423e | away_win 5-4 | correct |
| 824339 | 31de423e | away_win 10-7 | inconclusive (no-decision) |
| 824993 | 5703433e | home_win 11-2 | inconclusive (no-decision) |
| 823526 | c149433e | away_win 11-4 | inconclusive (no-decision) |
| 823932 | be49433e | home_win 3-0 | correct |

15 directional: 9 correct / 6 incorrect (matches the sweep's per-game predictions
exactly). Cohort v4 subset: 8/6 = 57.1%. 3 no-decision rows recorded honestly.

## post-settlement corpus readout (eras NEVER pooled; observation, not validation)

- settled non-excluded: 155 (directional 134 + no-decision 21); valid directional
  denominator 119 -> 134
- all directional: 78/56 = 58.2%
- by era: v1-era live/legacy n=82, 58.5% | v1 registry n=37, 56.8% | **v2 n=15, 9/6 =
  60.0% (UNCHANGED -- no v2 row settled today; the v2 sample is untouched)**
- market agreement: agree 51/85 = 60.0% vs disagree 4/9 = 44.4% -- the disagreement
  inversion persists at a still-small n=9; no market-superiority claim
- strength bands: High 63/113 (55.8%) vs Medium 15/21 (71.4%) -- inversion persists;
  numeric confidence remains anti-informative doctrine, unchanged
- lean split: home 54/96 (56.2%) vs away 24/38 (63.2%) -- home bias persists
- Gate 4: remains FALSE; nothing here authorizes tuning, model changes, or buyer claims

## state after this slice

0 identity-bearing reconciliation candidates remain; the only unsettled non-excluded
rows are the 102 identity-less legacy runs (blocked, unchanged) and 1 stale pending
run (087a433e, exclusion needs its own authorization). RC Gate 1 (2026-07-17 TB@BOS
DH 824766/824737) unaffected: drill identities unconsumed, caps untouched, runtime
cold. Posture remains no-spend; paid lanes remain not-authorized.
