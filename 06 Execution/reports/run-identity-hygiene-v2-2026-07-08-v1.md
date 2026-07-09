---
title: "Run Identity Hygiene v2: full duplicate-active remediation (17 gamePks -> 0)"
type: "evidence-report"
date: "2026-07-08"
status: "COMPLETE -- 19 operator-approved exclusions applied; duplicate-active surface eliminated"
project: "DAI"
slice: "Run Identity Hygiene v2"
repos:
  dai: "unchanged (a0db824)"
  dai-vault: "docs-only"
related:
  - "06 Execution/reports/run-identity-hygiene-2026-07-08-v1.md"
  - "04 Products/sports-v1/run-eligibility-and-supersession-contract-v1.md"
  - "06 Execution/reports/gate4-evidence-readout-backed-depth-capture-2026-07-07-v1.md"
---

# run identity hygiene v2 -- 17 remaining duplicate-active gamePks

## 1. objective

diagnose and safely remediate the 17 duplicate-active gamePks left after hygiene v1,
823613 first (two ACTIVE SETTLED runs with opposite leans, both in the valid-settled
denominator), with no /reconcile calls, no paid calls, no prompt/threshold/calibration/
buyer/capture changes, and no settlement of 823281.

## 2. starting state

- dai a0db824 synced, unchanged. dai-vault a941c2f pushed (the two v1 commits went to
  origin at slice start per operator instruction).
- devcore-sql up; DevCore.Api :5007 up (from v1); agent-service never started.
- BEFORE (fresh reads): 17 duplicate-active gamePks / 35 active rows; valid-settled
  122 = 61 correct / 43 incorrect / 18 inconclusive (directional 104, acc 0.5865);
  excluded runs 19; outcomes 124 / evaluations 124.

## 3. diagnosis (all 35 rows origin-traced)

- **823613 CHC@NYM (the priority): postponement double-settlement.** StatsAPI shows
  the 06-22 instance POSTPONED (rain, statusCode DR) and rescheduled to 06-24, final
  CHC 10 - NYM 3 (away_win). run 200018/beb5433e (06-22 pregame, lean away) priced the
  rained-out instance; run 220014/d879433e (06-24 pregame, lean home) priced the game
  as played. the 06-25 "directional-contrast settlement pass v1" wrote outcomes to
  BOTH (per-run, one second apart) -> beb5433e graded "correct" against a game it
  never predicted. that lucky-correct row was the denominator contamination.
- 12 gamePks (822795, 822959, 823037, 823204, 823362, 823686, 823769, 824011, 824256,
  824422, 824580, 824821): morning directional-contrast-cohort-v4 run (240xxx,
  ~13:40Z 06-28, authoritative) + evening real-cohort-live-soak-v1 capture run
  (2500xx, ~22:54Z), both unsettled. note 824011 had opposite leans (v4 away, soak
  home).
- 824744 NYY@BOS x3: v4 run 240027/7137433e (authoritative) + ecdd423e (the "+1
  prior" first-live-capture run from MLB Analyzer Request Capture v1) + 27de423e
  (registry canary backed-depth confirmation run).
- 824818 / 825066: unsettled starter-missing-regime-capture run + SETTLED soak run
  (28bd433e settled in follow-up v6; 25bd433e settled in the v7c-era pass) -- same
  shape as the 824662 fix in v1.
- 824990 LAA@ATH: BOTH active rows (09aa433e, 0daa433e, 10 minutes apart 06-19) are
  frontend visual-QA generations, logged at creation as "additional active MLB runs,
  not part of that cohort". neither was ever a calibration row.

## 4. remediation applied (operator approved the named 19 in-session)

via POST /api/agent-runs/{id}/exclude; every response echoed the stored state:

```text
diagnostic (16):
  f9dd423e 00de423e 12de423e 17de423e f6dd423e 03de423e 0fde423e 13de423e
  fbdd423e ffdd423e 0ade423e f3dd423e            (12 soak dupes, keep v4 runs)
  ecdd423e 27de423e                              (824744 capture + canary runs)
  09aa433e 0daa433e                              (824990 QA runs; gamePk now zero-active)
superseded (3):
  beb5433e -> d879433e                           (823613 postponed-instance run)
  3ade423e -> 28bd433e                           (824818)
  37de423e -> 25bd433e                           (825066)
```

not done, per constraints: no /reconcile, no settlement-row writes, no ledger entries,
no prompt/confidence/buyer/threshold/calibration/capture change, no 823281 settlement
(6a37433e untouched, still the single active run there).

## 5. before/after evidence (operator's verification checklist, all fresh reads)

| check                                             | before | after | verdict |
| ------------------------------------------------- | ------ | ----- | ------- |
| duplicate-active gamePks (db sweep)                | 17     | 0     | PASS |
| 823613 rows counted in valid set                   | 2 (opposite leans) | 1 (d879433e only; beb5433e superseded) | PASS |
| valid-settled n (/rows + db agree)                 | 122    | 121   | PASS -- moved ONLY via 823613 |
| valid eval distribution (correct/incorrect/inconcl)| 61/43/18 | 60/43/18 | PASS -- exactly the lucky-correct removed |
| valid directional n / accuracy                     | 104 / 0.5865 | 103 / 0.5825 | honest direction (accuracy DOWN) |
| outcomes / evaluations totals                      | 124/124 | 124/124 | PASS -- unchanged |
| excluded runs total                                | 19     | 38    | PASS -- +19 exactly |
| market-opposed valid-settled (disagreement ledger) | 6      | 6     | PASS -- untouched; n=7 checkpoint NOT fired |
| unrelated runs mutated                             | --     | none  | PASS -- excluded delta == the named 19 |

note: the permission layer initially denied the batch as not-individually-named; the
19 were then presented run-by-run and explicitly approved before application.

## 6. what this changes for the gate

valid-settled 122 -> 121 and directional accuracy 0.5865 -> 0.5825; the change removes
contaminated evidence rather than improving any metric. discrimination regions,
disagreement n (6), and failingReasons (discrimination_inverted,
insufficient_market_disagreement) are structurally unaffected; conclusionsAllowed
remains FALSE. no readout produced by this slice makes any accuracy claim.

## 7. remaining risks

- zero duplicate-active gamePks remain, but nothing PREVENTS recurrence: automatic
  supersession-at-generation is still contract-deferred, and any future
  capture/canary/QA batch re-running a real game recreates the hazard. cheapest
  guard: make evidence-run exclusion part of the capture-slice closeout checklist
  (mark diagnostic at creation time).
- 824990 now has zero active runs (both QA) -- identity reconcile for it returns
  NoMatch by design; harmless, documented.
- 823281 stands single-active on a DeliberateDivergence row; settling it creates the
  first deliberate ledger entry + fires the n=7 checkpoint (separate approval,
  still deferred per operator instruction).
- exclusion has no un-exclude endpoint; reversal would be a manual db/api addition.
  all 19 runs, artifacts, and traces are preserved (soft flags).

## 8. next recommended action

1. adopt the capture-closeout exclusion rule (docs-only pattern note) before the next
   evidence batch -- prevents hygiene v3.
2. decide 823281/6a37433e settlement (deliberate-ledger first entry; own approval).
3. Prompt Market Context Hardening v1 (approval-gated) -- identity surface is now
   clean enough to trust its before/after readouts.
