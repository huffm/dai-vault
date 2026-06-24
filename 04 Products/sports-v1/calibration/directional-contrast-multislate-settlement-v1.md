# Directional-Contrast Multi-Slate Settlement + Capture Continuation v1

**date:** 2026-06-24
**status:** WAIT-ONLY -- settlement gate NOT met. 0 games reconciled; no outcome/eval rows written. Corpus totals
unchanged 47/47. No tuning, no code/runtime change.
**type:** calibration + measurement (settlement-gated). Ran through dai-slice-runner + dai-skill-router gate +
dai-grill-with-vault + superpowers:verification-before-completion. No dai-code-reviewer (no code).

**Anchor:** Reconcile only Final games on authoritative outcomes (MLB StatsAPI); never guess a score. Presence of a
captured, identity-bearing cohort is not a settled outcome.

## Gate check (2026-06-24 ~17:26Z)

The cohort-v2 games (the 14 runs captured 2026-06-24) are tonight's slate and have NOT settled. StatsAPI state for
the 14 gamePks at check time:

- Scheduled (not started): 10
- In Progress: 2 (823850 Rangers@Marlins 1-1; 823613 Cubs@Mets 0-0)
- Pre-Game: 1 (824341 RedSox@Rockies)
- Delayed Start: 1 (824583 Guardians@WhiteSox)
- (stale Postponed listing for 823613 from the 06-22 reschedule; today's 823613 is the In Progress game)

First pitches are this afternoon/evening; games are not expected Final until ~2026-06-25T04:00Z+. The only
outstanding 200x-cohort game (200018 / beb5433e, Cubs@Mets, gamePk 823613) is the same game and is also only In
Progress. **Zero games are Final, so zero are reconcilable.** No outcomes posted, no rows written.

## Material preconditions (verified live)

| precondition | result |
|---|---|
| Directional-Contrast cohorts contain newly settled games | **FALSE** -- all In Progress / Scheduled / Pre-Game |
| Outcome reconciliation data available | **FALSE** -- StatsAPI has no Final scores for these games yet |
| Existing corpus valid and uncontaminated | TRUE -- AgentRunOutcomes 47 / AgentRunEvaluations 47, unchanged (sqlcmd) |
| No previously identified blocking defect | TRUE |
| Cohort artifacts + market baseline accessible | TRUE -- 14 cohort-v2 runs active/completed; market baseline recorded in directional-contrast-cohort-capture-v2.md |

Two of the five preconditions fail. The slice's stated dependency ("cohorts have reached settlement") does not hold
yet; this pass is therefore wait-only by design, not by error.

## Reconciliation results

None this pass. No `POST /api/agent-runs/{id}/outcome` submitted. No outcome row, no evaluation row, no in-progress
score posted, no outcome fabricated. Corpus stays 47/47.

## Performance / scarce-signal analysis

Cannot be produced this pass -- it requires settled outcomes, and there are none new. The pre-settlement structure
(captured at capture time, not an outcome): DAI's lean matched the market favorite on 12 of 13 decided-lean
cohort-v2 games; the lone disagreement is Brewers@Reds (DAI home, market away); DAI abstained (null) on
Braves@Padres. These are the scarce-signal games to grade once Final -- the disagreement game and the null
abstention carry the most information per the Read v2 analysis. No DAI-vs-market/home/chance results exist yet.

## Calibration readiness assessment

- **Calibration Read v3: NOT READY.** The settled corpus is unchanged at 47 (Read v2's basis). No new
  slate-balanced settled data has been added. Read v2's bar (>=3 pooled balanced slates) is unmet.
- **Additional settlement collection: REQUIRED and IMMINENT.** Rerun this settlement pass after ~2026-06-25T04:00Z+
  to reconcile the cohort-v2 14 + 200018 on authoritative finals.
- **Additional directional-contrast capture: YES, after settling this one** -- capture 1-2 more daily slates (with
  market baseline) so the disagreement sample grows; DAI~market agreement keeps disagreement games scarce (~1/slate).

## Buyer-safe findings

- Learned this pass: nothing new about performance -- the games are not yet played. The cohort and its market
  baseline are staged and fully reconcilable once Final; corpus integrity is intact.
- Remains uncertain: whether DAI has any directional edge beyond the market favorite (the central open question).
- Evidence still missing: settled outcomes for the balanced cohort, especially the DAI-vs-market disagreement and
  null-abstention games; at least 2-3 settled balanced slates before Read v3.
- No tuning recommended (no validated discrimination target; settlement is the binding constraint, not architecture).

## What did not change

No application source, no model/prompt/FastAPI, no schema, no confidence/lean/posture/source-depth/advertised-
strength logic, no reconciliation-matcher logic, no buyer copy, no Tool Gateway/source/probe/freshness/billing/auth/
tenant. No DB writes. Settled corpus unchanged 47/47.

## Next recommended slice

Rerun **Directional-Contrast Multi-Slate Settlement + Capture Continuation v1 -- settlement pass** after
~2026-06-25T04:00Z+ (cohort-v2 14 + 200018 reconciled on authoritative StatsAPI finals; measure DAI vs market/home/
chance; isolate the disagreement + null games), then capture the 2026-06-25 slate. Only after >=3 settled balanced
slates exist: Reconciled-Outcome Calibration Read v3. No tuning until then.
