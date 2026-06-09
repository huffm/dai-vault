# Fresh Buyer Artifact Validation v1

**date:** 2026-06-09
**status:** validation slice. docs + intentional calibration artifacts only. no prompt/buyer-copy/phrase-bank/confidence/posture/lean/signal/decision change, no source addition, no probe-refresh activation, no schema change, no cost-guardrail/pricing work, no .NET/FastAPI/Angular code change.
**scope:** generate a small fresh batch of buyer artifacts now that local generation works, and inspect each packaged unit for buyer credibility, repetition, safety, and completeness.

## Purpose

Now that the full local generation path is operable (Fresh Artifact Generation Operability Fix v1, Post-Credit Fresh Artifact Generation Smoke v1), validate freshly manufactured units off the sports assembly line. Primary question: across newly generated artifacts, does the buyer-facing output remain credible, safe, non-repetitive, appropriately cautious, and useful enough to keep hardening toward paid validation? This is a validation slice, not a fix slice.

## Scope

- start/verify local services (Docker, devcore-sql, FastAPI analyzer, .NET platform API)
- generate a small fresh batch (4-6) through the intended `.NET POST /api/agent-runs` full chain
- inspect each artifact JSON and its buyer projection together
- document buyer credibility, repetition, safety/toutiness, completeness
- record a cost-awareness note

## Non-goals

- no prompt, buyer-copy, or phrase-bank change
- no confidence/posture/lean/signal/decision-logic change
- no source addition, no probe-refresh activation, no schema change
- no cost-guardrail or pricing implementation
- no auth/billing/tenant/dashboard/Kubernetes/deployment work
- no Jera work, no push

## Service readiness summary

- Docker daemon up (server 29.1.3); `devcore-sql` Up, `0.0.0.0:1433->1433`.
- FastAPI analyzer `127.0.0.1:8000` -- `GET /api/ping` -> 200.
- .NET DevCore.Api `localhost:5007` -- `GET /health` -> 200.
- All four services healthy before generation.

## Sample set generated

Five fresh artifacts via the calibration harness (`scripts/dev/sports/run-artifact-calibration.ps1`), which drives the full `.NET POST /api/agent-runs` chain and writes per-artifact JSON plus a calibration report under the accepted convention `dai-vault/04 Products/sports-v1/calibration/`.

Slate availability shaped the sample:
- NBA: only 1 upcoming game in the next 14 days (Finals window) -- generated it.
- MLB: 15 upcoming games -- generated 4 (single same-day slate, 2026-06-09).

Limitations (not faked): only 1 NBA game was available; no naturally weak/no-signal case appeared (every MLB run grounded `starting_pitching`; the NBA run grounded `market` + `rest_schedule`); no legacy/coarse artifacts (all fresh v3). I did not force artificial cases -- there is no dev-harness convention to manufacture thin cases without source manipulation, which is out of scope.

5 model calls total (one `gpt-4o-mini` call per run).

## Generation method

`run-artifact-calibration.ps1 -Competition nba -Days 14 -Take 1 -Force` and `-Competition mlb -Days 7 -Take 4 -Force`. Each run: `.NET /api/agent-runs` -> FastAPI `/api/sports/analyze` (one model call) -> .NET evaluate/quality_check/compose -> SQL persist -> artifact fetched via `GET /api/agent-runs/{id}/artifact`.

## Artifact table

All five: `artifactVersion = sports_decision_artifact_v3`, `cognitiveProtocol` present, `status = completed`, buyer projection present, persisted to SQL.

- NBA Spurs at Knicks (2026-06-10), run 9b6b433e: posture monitor, confidence 0.675 (analyzer 0.75), evidenceRichness 2, grounded `rest_schedule`+`market`, missing `sharp_public`, retrieve Degraded, no quality warnings.
- MLB Mariners at Orioles (2026-06-09), run 9f6b433e: posture monitor, confidence 0.75, evidenceRichness 1, grounded `starting_pitching`, quality_check Degraded (3 ungrounded-signal warnings: bullpen, lineup_form, ballpark).
- MLB Diamondbacks at Marlins, run a46b433e: posture monitor, confidence 0.75, evidenceRichness 1, grounded `starting_pitching`, quality_check Succeeded, no warnings.
- MLB Red Sox at Rays, run a56b433e: posture monitor, confidence 0.75, evidenceRichness 1, grounded `starting_pitching`, quality_check Degraded (bullpen, lineup_form, ballpark warnings).
- MLB Yankees at Guardians, run aa6b433e: posture monitor, confidence 0.75, evidenceRichness 1, grounded `starting_pitching`, quality_check Succeeded, no warnings.

Natural diversity achieved: one measured-confirmation case (NBA, market grounded but sharp_public missing, confidence dampened 0.75 -> 0.675, explicit "market-only lean"); two clean single-signal leans (Diamondbacks/Gallen, Yankees/Cole); two thinner home-field leans with internal Degraded quality_check (Orioles, Rays). No weak/no-signal case occurred.

## Buyer credibility findings

Pass. Every unit reads as measured decision support, not a tout sheet.

- Direction is plain and hedged: "Slight lean toward New York Knicks based on market favoring them by 2.5 points"; "Slight lean toward Yankees based on starting pitching advantage."
- The NBA measured case is appropriately cautious: "Confirmation is not strong enough to support a firmer stance, so this is a market-only lean."
- Each lean is linked to a basis (market spread, rest, or the starting-pitching matchup with named pitchers).
- All five hold `monitor` posture -- no aggressive certainty.
- No fake precision: the only numbers are grounded (NBA 2.5-point spread from market data; MLB named probable starters).

One soft watch: two MLB summaries reference factors that were not grounded -- "potential bullpen depth could play a significant role" (Orioles) and "both teams' bullpens and lineup forms will play a crucial role" (Rays). These are hedged ("potential", "will play a role"), not asserted facts, and the system caught them internally via `quality_check: Degraded` with explicit "signals_used contains bullpen but bullpen was not grounded" warnings. They are not buyer-visible factory terms and not tout language, but they are the model gesturing at data it did not ground. This is a grounding/prompt-quality watch, not a credibility failure.

## Repetition findings

Controlled consistency; no harmful or betting-adjacent sameness.

- "Slight lean toward" appears 5/5 -- the structural lean template, by design (controlled product consistency, same as prior passes).
- MLB lean bases split: "starting pitching advantage" (2/4) and "home field advantage" (2/4) -- two variants, not one robotic line.
- Generic hedge phrasing recurs on the MLB batch: counter_case "could outperform"/"could outperform expectations" (2/4), and what_would_change filler such as "recent form" (2/4). These are model-laziness-adjacent, not buyer-trust-damaging, and the deterministic harness flagged them (`counter_case_generic` 2/4, `what_would_change_contains_filler` 2/4).
- watch_for follows an "any significant ... changes" pattern across the MLB runs -- mildly robotic on a single same-day slate, but watch_for is inherently a "what to monitor" field; acceptable.

Judgment on repetition: borderline but not yet phrase-bank-justifying on a 4-game single-day MLB sample. The generic counter_case/what_would_change phrasing is a real recurring pattern worth watching; it is the MLB calibration report's own deterministic basis for recommending Cognitive Prompt Tightening v1.5 (prompt-quality flags on 3 of 4 runs). Surfaced here for the human to weigh; not actioned in this validation slice.

## Safety / toutiness findings

Pass. Explicit scan over the buyer-facing fields (`lean`, `summary`, `counterCase`, `watchFor`, `whatWouldChangeTheRead`) of all five artifacts returned 0 unsafe hits.

- no lock/guarantee/must-bet/hammer/free-money/easy-money/can't-miss/slam/unit/best-bet/sharp-side/edge-toward/value-play language.
- no aggressive posture leaked into buyer copy (all postures `monitor`).
- internal guardrail/permission language (e.g. `block_aggressive_posture`, `aggressive_blocked`) appears only in internal `signalAvailability` fields, never in buyer copy.

## Artifact completeness findings

Pass.

- decision/read appears first (`lean` + `summary`), supporting basis follows.
- uncertainty is visible without paralysis (`counterCase`, `monitor` posture, NBA "not strong enough").
- weak/thin runs still feel honest and useful (direction + counter + watch even on single-signal MLB).
- no empty or dangling buyer fields across the five units; all buyer fields populated.
- rich v3 `signalAvailability` handled correctly (NBA: market + rest grounded, sharp_public missing with follow-up signals listed).
- buyer projection aligns with posture/confidence (monitor + slight lean is internally consistent).

Calibration watch (not a packaging defect): all 4 MLB runs carry confidence 0.75 with evidenceRichness 1, i.e. `confidence_high_for_partial_evidence` on 4/4. The buyer-facing posture stays cautious (`monitor`), but the confidence number is high relative to the single grounded signal. This confirms the prior quality pass's MLB single-signal-high-confidence watch with fresh data.

## Cost awareness note

- model calls made: 5 (one `gpt-4o-mini` call per run; sports generation has exactly one model call per run).
- artifacts generated: 5 (1 NBA, 4 MLB).
- token/cost data: unavailable. `response.usage` is still not captured, so no per-call prompt/completion/total tokens, cost, or latency were recorded. The calibration harness prints only a heuristic ~$0.01-0.03/run estimate, not actual usage.
- reminder: Artifact Cost Guardrails v1 should instrument the single analyzer model call (usage, cost, latency, model, retry) before any larger calibration batches.

## Pass/fail judgment

**Pass with minor follow-up.**

Fresh buyer artifacts are credible, safe, appropriately cautious, and useful. The buyer copy itself needs no immediate change. The follow-ups are real but minor and known: (1) MLB single-signal confidence sits at 0.75 with evidenceRichness 1 on 4/4 fresh runs (a confidence-calibration watch); (2) generic counter_case/what_would_change phrasing on ~half the MLB batch (a prompt-quality watch the harness escalates); (3) two MLB summaries reference ungrounded bullpen/lineup_form factors in hedged language (a grounding watch, already caught internally). None of these is a buyer-facing safety or copy emergency.

## Recommended next slice

Primary: **Confidence Calibration Rules v1** -- the strongest and most universal fresh finding is `confidence_high_for_partial_evidence` on 4/4 MLB runs (0.75 confidence on one grounded signal, evidenceRichness 1), now observed across two consecutive passes. This is a confidence/evidence-fit question, deliberately out of scope to change in this slice.

Secondary candidates (human to weigh):
- **Cognitive Prompt Tightening v1.5** -- the MLB calibration report's own deterministic recommendation for the generic counter_case/what_would_change phrasing (prompt-quality flags on 3/4). Tighten only if the pattern persists on a larger fresh batch.
- **Signal Follow-Up Diagnostics v1** -- the NBA calibration report's recommendation (sharp_public missing, line_movement not implemented), consistent with existing deferrals.
- **Artifact Cost Guardrails v1** -- should precede larger calibration batches so model spend is metered.

## Deferred decisions / ledger

No ledger update needed. No new deferred runtime/prompt/source/artifact-contract decision was discovered. Existing deferrals (probe-refresh dormant entries 1-5; sharp_public/line_movement follow-up coverage) are unchanged. Confidence calibration and prompt tightening remain recommendation candidates, not committed seams.

## What was not changed

- no FastAPI prompt, route, or model code
- no parser / sanitation behavior
- no .NET runtime logic, controller, evaluator, or composer
- no confidence/posture/lean/signal/decision/artifact-semantics change
- no phrase bank, no buyer copy change
- no schema or migration
- no source expansion, no probe-refresh activation
- no Angular code
- no cost-guardrail, metering, or pricing implementation
- no auth/billing/tenant/dashboard/Kubernetes/deployment work
- no Jera work
- no `OPENAI_API_KEY` value change and no secret written to the repo

## Validation performed

- service health: FastAPI ping 200; .NET health 200; Docker 29.1.3; devcore-sql up on 1433.
- generation: 5 runs, all `status: completed`, all persisted, all `sports_decision_artifact_v3` with `cognitiveProtocol`.
- buyer-copy safety scan: 0 unsafe hits across 5 (lean/summary/counterCase/watchFor/whatWouldChangeTheRead).
- deterministic harness flags reviewed (NBA and MLB calibration reports).
- docs + artifacts checks: `git diff --check`, added-line exact-path scan, vault non-ASCII added-line scan.
