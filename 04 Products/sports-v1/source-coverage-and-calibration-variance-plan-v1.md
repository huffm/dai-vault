# Source Coverage and Calibration Variance Plan v1

**date:** 2026-06-17
**status:** plan / architecture-control. Docs-only. No code, no model, no generation, no reconciliation, no advisory/enforcement, no Probe, no source integration, no migration.
**classification:** source-coverage and calibration-variance strategy for the sports decision factory.

**Anchor:** Outcome calibration requires source variance; otherwise we are only measuring one bucket repeatedly.

## 1. Problem statement

The first reconciled MLB cohort produced outcome evidence (9 runs, 7/9 correct) but **zero control-signal variance**: all 9 were `thin` / `FulfilledWithThinCoverage` / `thin_sport_critical_grounded`. A predictor with one value across the whole sample cannot explain outcome risk. The bottleneck is not outcome volume -- it is *evidence-state variance*. This plan defines which source states we must create, how, and in what order, so outcomes can start teaching us which evidence states are reliable, risky, or non-actionable -- without building advisory/enforcement authority prematurely.

## 2. Principal engineer review

1. **What did the 9-run reconciliation prove?** That observed thin-fulfilled MLB artifacts *can* be directionally useful in a small cohort. An existence result, not a reliability result.
2. **What did it fail to prove, because all samples were thin?** Anything comparative: whether thin is worse than moderate/rich, whether any negative state predicts failure, whether the band carries information. With one band, correlation to outcome is undefined.
3. **Which source groups create moderate/rich band variance?** Band is driven by the count of grounded decision-useful *signals* (`SourceSufficiencyBuilder.DeriveBand`: 0 -> insufficient, 1 -> thin, >=2 -> moderate, and >=2 plus sport-critical grounded plus >=2 grounded supporting/contextual groups -> rich). `identity_schedule` comes from the run row and does **not** count. MLB grounds exactly one signal today (`starting_pitching`), so it is pinned at thin. Moving to **moderate** requires any second grounded signal; **rich** requires `starting_pitching` plus >=2 grounded supporting/contextual groups (e.g. bullpen + weather, or lineup + rest).
4. **Which source groups create negative readiness-state variance?** Per `SportSufficiencyProfiles[mlb]` (required `[starting_pitching]`, `AllowPrimaryFulfillment=true`, `AllowProbeFallback=false`) and `PerceiveFulfillmentPolicy`:
   - `PrimaryFulfillmentRequired` -- fires when `starting_pitching` is ungrounded but its supported primary path exists (pre-announcement). **Already observable in MLB**, no new source.
   - `NoDirectionalSeparation` -- grounded but null lean (the 824993 case). **Already observable in MLB**, no new source.
   - `ProbeRequired` -- requires `AllowProbeFallback=true`; MLB has it false. **Not producible without enabling probe fallback (deferred).**
   - `BlockedNotEvaluable` -- requires a required group with no supported path; MLB's required group always has one. **Not producible in MLB.**
5. **Can MLB alone produce enough variance?** Partly. MLB can produce thin, `PrimaryFulfillmentRequired`, and `NoDirectionalSeparation` now (free, via sample timing), and moderate after one source integration. It cannot produce `ProbeRequired`/`BlockedNotEvaluable` (those need probe activation or a different sport profile) -- and those stay deferred regardless.
6. **Highest signal-per-effort sport/source combos?** In June, MLB is the only in-season buyer-ready sport (NBA finals ~over and otherwise offseason; NFL/NCAAF/NCAAMB out of season; WNBA in-season but unsupported/deferred). So near-term variance must come from within MLB. The single cheapest real source is **MLB market odds via odds-api baseball** -- same provider and key already wired for NBA/football (`MarketBasketballSpread`/`MarketFootballSpread`), reliability high, cost = existing odds-api quota.
7. **Which integrations create product value, not just technical completeness?** Market odds (consensus line is the strongest single sports signal and the clearest buyer explanation: "our lean agrees/disagrees with the market"); lineup/injury (who is actually playing); weather/park for totals. Team-form and rest/travel add little for MLB.
8. **What would be dangerous to build before variance exists?** Advisory/enforcement plumbing, buyer track-record surfaces, confidence-threshold changes, or probe execution -- any authority built on a single zero-variance band. Also dangerous: integrating sources that do not change a decision, an explanation, or a sufficiency state (cost without calibration value).
9. **What should remain deferred?** Advisory, soft/hard enforcement, buyer display, live Probe execution, structured Question trace, tenant/product overrides, WNBA implementation, multi-sport expansion until those sports are in season.
10. **Smallest next implementation slice that increases calibration value?** MLB market-odds grounding (thin -> moderate band variance + buyer value), paired with a **no-code** sample-design change that captures `PrimaryFulfillmentRequired` and `NoDirectionalSeparation` via generation timing.

## 3. Product architect review

Source value is judged by whether a source changes a **decision**, an **explanation**, or a **sufficiency state** -- not by data availability.

| Source group | Improves decision? | Improves buyer explanation? | Reduces null/fragile leans? | Distinguishes thin vs moderate/rich? | Visible buyer advantage? | Reliable/affordable? | Reusable across sports? | Worth before advisory? |
|---|---|---|---|---|---|---|---|---|
| market_odds | Yes (consensus baseline) | Yes (agree/disagree vs line) | Yes (line always exists pre-game) | Yes (2nd signal -> moderate) | Yes | Yes (odds-api, already paid) | Yes (all sports) | **Yes** |
| lineup_injury | Yes (who plays) | Yes | Some | Yes | Yes | Medium (statsapi lineups, timing) | Yes | Next |
| weather_park | Some (totals/park) | Yes | Little | Yes | Some | Medium (new weather provider) | MLB-centric | Later |
| bullpen_availability | Some | Some | Little | Yes | Some | Medium (no current source) | MLB-centric | Later |
| market_movement | Some (sharp/public) | Yes | Little | Yes | Some | Medium (actionnetwork, MLB coverage TBD) | Yes | Later |
| team_form | Little (priors-ish) | Some | Little | Yes (contextual) | Little | Medium | Yes | Defer |
| rest_travel | Little for MLB | Some | Little | Yes (contextual) | Little (MLB) | Free (espn, NBA only today) | NBA-centric | Defer (MLB) |
| identity_schedule | n/a (already grounded) | Yes | n/a | No (does not count to band) | n/a | Yes | Yes | Done |
| starting_pitching | n/a (already grounded) | Yes | n/a | No (it is the thin anchor) | Yes | Yes | MLB | Done |
| fallback_proxy | No (taxonomy bucket) | No | No | No | No | n/a | n/a | Never primary |

## 4. Current evidence summary

- outcomes/evaluations: 21 / 21 (DB-verified).
- reconciled MLB cohort: 9 active usable runs (7 original + 2 rerun), all `LeanSide=home`.
- result: 7 correct / 2 incorrect / 0 inconclusive = 77.8%.
- evidence state: **all 9** `thin` / `FulfilledWithThinCoverage` / `thin_sport_critical_grounded`, grounded `[identity_schedule, starting_pitching]`, no missing critical, no null reason. Standing decision (calibration review v1): observed-only live, no advisory, no enforcement.

## 5. Why current calibration lacks predictor variance

The predictor (PerceiveFulfillment decision / SourceSufficiency band) is **constant** across the entire reconciled sample. A constant cannot correlate with anything: it labeled a 12-0 win and a 3-9 loss identically. Growing n in the same bucket multiplies confidence in one cell of a table that has only one cell. Calibration needs the predictor to **vary** against outcome -- different bands and different decision states, each with enough settled outcomes to compare hit rates.

## 6. Source coverage matrix (MLB focus, current truth from code)

Current support and grounding from `ProbeFallbackCatalog` + `SourceSignalTaxonomy` + `SportSufficiencyProfiles[mlb]`.

| Source group | MLB support today | MLB grounded today | Band-variance lever | Negative-state lever | Product value | Calibration value | Effort | Reliability risk | Cost/rate risk | Priority | Next slice |
|---|---|---|---|---|---|---|---|---|---|---|---|
| identity_schedule | supported (statsapi) | yes (run row) | no (excluded from band) | no | high | low | done | low | none | done | -- |
| starting_pitching | supported (statsapi) | yes (1 signal) | no (the thin anchor) | yes (absence -> PrimaryFulfillmentRequired) | high | high | done | low | none | done | -- |
| market_odds | future_candidate (odds-api, paid, key) | no | **yes (thin -> moderate)** | no | high | high | **low** (provider+key wired) | low | low (existing quota) | **NOW** | MLB Market Odds Grounding v1 |
| lineup_injury | future (no source) | no | yes (-> moderate/rich) | no | high | medium | medium | medium | low | next | MLB Lineup/Injury Grounding v1 |
| weather_park | future (no source) | no | yes (contextual -> rich) | no | medium | medium | medium-high (new provider) | medium | low-med | later | MLB Weather/Park Grounding v1 |
| bullpen_availability | future (no source) | no | yes (-> moderate/rich) | no | medium | medium | medium-high | medium | low | later | -- |
| market_movement | future for MLB (actionnetwork = NBA) | no | yes (supporting) | no | medium | medium | medium | medium | low | later | -- |
| team_form | future (priors only) | no | yes (contextual) | no | low | low-med | medium | medium | low | defer | -- |
| rest_travel | future for MLB (espn = NBA) | no | yes (contextual) | no | low (MLB) | low (MLB) | medium | medium | none | defer (MLB) | -- |
| fallback_proxy | unsupported (bucket) | no | no | no | none | none | n/a | n/a | n/a | never primary | -- |

## 7. Sport-by-sport variance opportunity

| Sport | In season (June 2026) | Buyer-ready | Supported groups | Near-term variance value |
|---|---|---|---|---|
| MLB | yes | yes | identity, starting_pitching (+ market_odds one slice away) | **primary near-term source of variance** |
| NBA | no (finals ~over, then offseason) | yes | identity, market_odds, rest_travel, market_movement | high variance potential, blocked by season until ~Oct |
| NFL | no (starts Sep) | smoke only | identity, market_odds, market_movement | revisit Sep; needs buyer validation |
| NCAAF | no (starts Aug/Sep) | smoke only | identity, market_odds, market_movement | revisit fall |
| NCAAMB | no (starts Nov) | smoke only | identity, market_odds, rest_travel, market_movement | revisit Nov |
| WNBA | yes | no (unsupported, deferred) | none | deferred (entry 26); supportability analysis only |

Implication: multi-sport variance (NBA's four supported groups) is the richest medium-term lever but is **season-gated**. Until fall, MLB is the only live engine, so the near-term plan is MLB-internal.

## 8. MLB source expansion recommendation

**Add `market_odds` first** (odds-api baseball), over bullpen/weather/lineup/team_form, because it wins on all four axes:
- **calibration variance:** cleanly moves MLB from thin (1 signal) to moderate (2 signals); produces the first non-thin reconciled cohort and a new comparison cell (does line agreement/disagreement with the lean predict correctness?).
- **product value:** the market line is the single most informative sports signal and the clearest buyer explanation.
- **implementation feasibility:** lowest -- odds-api and the key are already integrated and paid for NBA/football via `MarketBasketballSpread`/`MarketFootballSpread`; baseball is another sport key on the same client/grounding path, mapped to the existing `market` signal that `SourceSignalTaxonomy` already classifies as `market_odds` (Critical).
- **source risk:** low and known (existing dependency); cost is existing odds-api quota.

**Should we collect more thin-only MLB first?** No. More thin-only samples measure the same bucket (the anchor problem). **Expand source coverage AND, in parallel, capture negative states via timing** (section 9) -- both add variance; piling thin does not.

## 9. Calibration sample design

**"Enough variance" for the next stage** = the predictor takes multiple values, each with enough settled outcomes to compare hit rates. These are directional design targets, not power-calculated thresholds (single sport, single season -- treat as a first read, not a calibrated model).

- **Minimum settled outcomes per observed state:** target >=15-20 per state for a first directional comparison; >=30 before any advisory consideration. Below ~15 a per-state hit rate is anecdote.
- **States to capture and how:**
  - `FulfilledWithThinCoverage` / thin -- have ~9; keep accumulating (free, every standard MLB run).
  - **moderate** -- requires market_odds grounding (section 8); not attainable until that slice ships.
  - **rich** -- requires starting_pitching + >=2 grounded supporting/contextual groups; attainable only after a second/third MLB source (lineup/weather). Later.
  - `PrimaryFulfillmentRequired` -- **capturable now, no code:** generate MLB runs before starters are announced and carry them to settlement (do not supersede them). The 823046/822887 pre-announcement reads already demonstrated this state; they were superseded before settlement, so none was reconciled in that state.
  - `NoDirectionalSeparation` -- **capturable now:** reconcile grounded-but-null-lean games (e.g. the 824993 family) as comparison cases.
  - `ProbeRequired` / `BlockedNotEvaluable` -- deferred (need probe activation or an unsupported-required-group sport); do not engineer for them yet.
- **Null-lean artifacts:** reconcile as comparison cases. They evaluate `inconclusive` (no directional winner), and they are the only way to populate `NoDirectionalSeparation`; track them as a distinct group, never folded into the directional hit rate.
- **Superseded artifacts:** stay excluded from active calibration (Run Eligibility contract: `ExclusionReason != null` is skipped by the matcher). Preserved for audit, never an active candidate.
- **Rerun cohorts:** tracked separately as an as-of cohort (as already done for 5003433e/5403433e); re-run timing changes the grounded state, so mixing them into the original cohort hit rate is contamination.
- **Avoiding lookahead contamination:** freeze identity and lean at generation; generate strictly before settlement; never regenerate or re-lean after seeing a result; reconcile only through the matcher on final scores; never let an outcome influence an artifact or a profile. The settlement gate (Final-only, no in-progress scores) stays mandatory.

## 10. What source states must be captured before advisory

Before even *internal* advisory can be evaluated (Option B from calibration review v1, reserved for negative readiness states), we need reconciled real-data outcomes for the states advisory would act on:
- a populated **moderate** (and ideally rich) band cohort to compare against thin;
- a reconciled **PrimaryFulfillmentRequired** cohort (does "primary evidence missing" actually predict worse outcomes?);
- a reconciled **NoDirectionalSeparation** cohort (does grounded-but-null behave differently from thin-fulfilled?).
Only when these cells exist with >=15-20 settled outcomes each can a `PerceiveFulfillment`-vs-outcome correlation be computed and advisory authority argued. Until then: observed-only, neutral.

## 11. What remains deferred

advisory mode; soft/hard enforcement; buyer-facing display / track record (ledger entry 12); live Probe execution and `ProbeRequired` capture; structured Question trace; tenant/product overrides; WNBA implementation (entry 26); multi-sport expansion (NBA/NFL/NCAAF/NCAAMB) until those sports are in season and buyer-validated; team_form and MLB rest_travel sources; confidence-threshold changes (entry 12).

## 12. Recommended next slice (concrete scope and boundaries)

**Decision: Option A -- MLB Market Odds Grounding v1**, paired with an immediate no-code negative-state capture in the next budgeted generation.

**MLB Market Odds Grounding v1 (implementation slice, scoped):**
- **In scope:** wire odds-api baseball into the existing market-grounding path so an MLB run grounds a `market` signal (-> `market_odds`, Critical) alongside `starting_pitching`, lifting grounded runs to the **moderate** band. Reuse the existing `OddsMarketClient`/grounding/identity plumbing already used for NBA/football; add the baseball sport key and map to the existing `market` signal category. TDD; focused tests then full `DevCore.Api.Tests`; final-verification build.
- **Out of scope / boundaries:** no advisory, no enforcement, no buyer-surface change, no confidence/posture/lean/threshold change, no Probe execution, no Tool Gateway authority change, no new profile enforcement mode, no other sport, no WNBA, no migration unless a column is genuinely required (justify first). Market odds is grounding/evidence only -- it must not silently become a lean driver or a buyer claim in this slice.
- **Outcome:** the first MLB runs capable of grounding two signals -> first **moderate**-band cohort -> first real band variance for calibration.

**Companion (no-code, in the next generation slice, not this one):** time generation to also capture `PrimaryFulfillmentRequired` (pre-announcement runs carried to settlement, not superseded) and reconcile null-lean games for `NoDirectionalSeparation`. This adds decision-state variance at zero integration cost.

Recommended sequence after that: MLB Lineup/Injury or Weather/Park grounding (toward rich), then revisit NBA multi-sport variance when the season returns (~Oct). No advisory/enforcement until the band and negative-state cells are populated and compared.
