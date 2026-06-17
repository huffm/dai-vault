# PerceiveFulfillment-vs-Outcome Calibration Review v1

**date:** 2026-06-17
**status:** complete -- read-only calibration review. No code, no model calls, no generation, no reconciliation, no advisory/enforcement, no Probe, no migration. Docs-only.
**classification:** calibration review on the first reconciled thin-fulfilled MLB cohort (n=9).

**Anchor:** A control signal earns authority by explaining outcome risk, not merely by existing.

## 1. Problem statement

The 9 active directionally-usable MLB runs are now reconciled (7 correct / 2 incorrect / 0 inconclusive, 77.8%). This review joins each run's outcome/evaluation to its observed-mode `PerceiveFulfillment` decision and `SourceSufficiency` band to decide what the Evidence Sufficiency Control Plane has actually proven, and whether observed mode has earned advisory or enforcement authority. Read-only; no promotion is executed here.

## 2. Principal engineer review

1. **What does 7/9 correct prove?** Only that observed thin-fulfilled MLB artifacts *can* produce directionally useful leans in this small cohort. It is an existence result, not a reliability result.
2. **What does it not prove?** That thin coverage is generally reliable; that advisory/buyer display/enforcement are justified; that moderate/rich, ProbeRequired, or BlockedNotEvaluable states are calibrated; that missing-source states predict failure; that the system is calibrated across sports; that NBA/WNBA/other profiles are ready.
3. **Is `FulfilledWithThinCoverage` too broad to support advisory by itself?** Yes. It fired identically on all 9 runs -- every correct AND every incorrect one. A label with zero variance across outcomes carries no advisory information.
4. **Did the two incorrect outcomes reveal a sufficiency failure?** No. Both incorrect runs are projection-identical to the seven correct: same band (thin), same decision, same grounded groups `[identity_schedule, starting_pitching]`, no missing critical groups, no null reason. The misses are home favorites losing -- normal sports/model directional variance, not a detectable evidence gap.
5. **Justify keeping observed mode live?** Yes. It is accurate, deterministic, derive-on-read, non-invasive (mutates no lean/confidence/outcome), and zero marginal cost. Keep collecting.
6. **Justify internal advisory mode?** No -- not for `FulfilledWithThinCoverage`. A constant caution on every run is noise, not signal.
7. **Justify buyer-facing advisory?** No. n=9, single band, no discrimination; buyer-visible track record (ledger entry 12) stays gated.
8. **Justify soft/hard enforcement?** No. Enforcement requires a calibrated negative state that predicts failure; none exists yet.
9. **What more evidence is needed?** Real reconciled outcomes for the *negative readiness* states (`PrimaryFulfillmentRequired`, `ProbeRequired`, `BlockedNotEvaluable`) and for moderate/rich bands, plus a larger n -- so the predictor actually varies against outcome.
10. **Overengineering to avoid:** building advisory/enforcement plumbing, buyer track-record surfaces, or threshold changes off a single zero-variance band; treating 77.8% as a calibrated number; productizing a caution that explains nothing.

## 3. Cohort definition

9 reconciled active runs (7 original active + 2 rerun active), all `LeanSide=home`. Holdouts not in this review: `5703433e` (active null lean, comparison only), `2e03433e` / `3603433e` / `4203433e` (superseded). Run eligibility unchanged (10 active / 3 superseded).

## 4. Per-run table

PF/SS captured read-only from `GET /api/agent-runs/{id}/artifact`; outcome/eval from the DB rows.

| # | run | gamePk | cohort | teams | lean | final (H-A) | eval | band | null reason | PF decision | PF reason | grounded | missing critical | read |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 1 | 2303433e | 823452 | original | Marlins @ Phillies | home | 7-0 | correct | thin | none | FulfilledWithThinCoverage | thin_sport_critical_grounded | identity_schedule, starting_pitching | none | normal variance |
| 2 | 2803433e | 822724 | original | Royals @ Nationals | home | 7-3 | correct | thin | none | FulfilledWithThinCoverage | thin_sport_critical_grounded | identity_schedule, starting_pitching | none | normal variance |
| 3 | 2a03433e | 824505 | original | Mets @ Reds | home | 12-0 | correct | thin | none | FulfilledWithThinCoverage | thin_sport_critical_grounded | identity_schedule, starting_pitching | none | normal variance |
| 4 | 3403433e | 824666 | original | Rockies @ Cubs | home | 5-4 | correct | thin | none | FulfilledWithThinCoverage | thin_sport_critical_grounded | identity_schedule, starting_pitching | none | normal variance |
| 5 | 3d03433e | 824181 | original | Tigers @ Astros | home | 3-9 | **incorrect** | thin | none | FulfilledWithThinCoverage | thin_sport_critical_grounded | identity_schedule, starting_pitching | none | normal variance (home fav lost; no sufficiency flag) |
| 6 | 4103433e | 825071 | original | Angels @ Diamondbacks | home | 4-3 | correct | thin | none | FulfilledWithThinCoverage | thin_sport_critical_grounded | identity_schedule, starting_pitching | none | normal variance |
| 7 | 4903433e | 823938 | original | Rays @ Dodgers | home | 4-3 | correct | thin | none | FulfilledWithThinCoverage | thin_sport_critical_grounded | identity_schedule, starting_pitching | none | normal variance |
| 8 | 5003433e | 823046 | rerun | Padres @ Cardinals | home | 3-0 | correct | thin | none | FulfilledWithThinCoverage | thin_sport_critical_grounded | identity_schedule, starting_pitching | none | normal variance |
| 9 | 5403433e | 822887 | rerun | Twins @ Rangers | home | 2-4 | **incorrect** | thin | none | FulfilledWithThinCoverage | thin_sport_critical_grounded | identity_schedule, starting_pitching | none | normal variance (home fav lost; no sufficiency flag) |

Missing *recommended* (non-critical) groups are also identical across all 9: `market_odds`, `bullpen_availability`, `lineup_injury`, `weather_park`, `market_movement` (MLB grounds only identity + starting pitching today). No grounded/missing difference separates the 2 incorrect from the 7 correct.

## 5. Result summary

- correct: 7 / 9
- incorrect: 2 / 9
- inconclusive: 0 / 9
- hit rate: 77.8%
- n = 9, single observed band (thin), single decision (FulfilledWithThinCoverage)

## 6. Original vs rerun summary

- original active (7): 6 correct / 1 incorrect = **85.7%**.
- rerun active (2): 1 correct / 1 incorrect = **50.0%** (n=2; not interpretable).
- The two rerun runs (originally null-lean, re-run once starters were announced) both grounded the same thin profile; the rerun split is noise at n=2, not a cohort effect.

## 7. What FulfilledWithThinCoverage explained

It correctly described the *evidence state*: every active MLB run grounds exactly the two critical groups (identity + starting pitching) and nothing else, so "thin but sport-critical grounded" is an accurate, deterministic read. As a *descriptor of coverage*, it is right on all 9.

## 8. What it did not explain

It explained **none of the outcome risk**. The label is constant across the sample, so its correlation with correctness is undefined (a zero-variance predictor cannot discriminate). It assigned the same reading to a 12-0 win and a 3-9 loss. On this cohort `FulfilledWithThinCoverage` has zero discriminating power over directional correctness, and the two misses showed no evidence gap it could have flagged.

## 9. Should observed mode remain live?

Yes. Accurate, non-invasive, deterministic, free. It is the data-collection substrate for any future calibration. No reason to disable.

## 10. Is advisory mode justified?

No.
- **Not for `FulfilledWithThinCoverage`** (option C rejected): constant across outcomes -> a caution on every run is noise.
- **Not buyer-facing** (option D rejected): entry 12 buyer track record stays gated at n=9.
- **Negative-state advisory (option B)** is the right *future shape* but is **premature now**: there are zero reconciled real-data examples of `PrimaryFulfillmentRequired` / `ProbeRequired` / `BlockedNotEvaluable` (those branches are unit-tested only; the only PrimaryFulfillmentRequired artifacts were the pre-announcement superseded nulls 823046/822887, never reconciled in that state). You cannot validate an advisory you have never seen fire against an outcome.

## 11. Is enforcement justified?

No (options E and harder rejected). Enforcement requires a calibrated state that predicts failure; nothing in this sample predicts anything. Soft and hard enforcement stay deferred.

## 12. Recommended promotion decision

**Option A -- keep observed-only for all states**, with a documented intent toward Option B (internal advisory reserved for the negative readiness states) *only after real reconciled examples of those states exist*. Do not advise on `FulfilledWithThinCoverage`. Do not buyer-display. Do not enforce. Keep reconciling outcomes to grow n and -- critically -- to capture band/decision *variance* (moderate/rich, and the negative states), without which calibration is impossible.

## 13. What remains deferred

- internal advisory mode (not justified at n=9, zero-variance band) -- revisit under Option B after negative-state examples exist.
- buyer-facing advisory / track record (entry 12, gated).
- soft / hard enforcement.
- ProbeRequired / BlockedNotEvaluable real-data validation.
- moderate / rich band real-data validation (MLB grounds at most one supporting signal today -> only thin is exercised live).
- structured Question trace (no deterministic decision-uncertainty producer yet).
- live Probe execution.
- tenant / product overrides.
- WNBA support (entry 26).

## 14. Recommended next slice

**Calibration Variance Capture Plan v1** (docs/planning, no spend in the planning slice): define how to obtain reconciled outcomes for non-thin and negative-readiness states so the predictor varies against outcome -- e.g. budgeted generation of runs expected to ground more (or fewer) signals, and deliberate capture of pre-announcement `PrimaryFulfillmentRequired` reads carried through to settlement. Only with a varying predictor can a real `PerceiveFulfillment`-vs-outcome correlation be computed and Option B evaluated. Until then, observed mode stays live and neutral. No code, no model spend in the planning slice.
