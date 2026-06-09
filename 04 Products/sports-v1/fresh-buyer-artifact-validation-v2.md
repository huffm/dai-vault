# Fresh Buyer Artifact Validation v2

**date:** 2026-06-09
**status:** validation slice. docs + intentional calibration artifacts only. no code change in any repo. no prompt/model-call/cost-guardrail/source/probe-refresh/lean/raw-confidence/confidence-constant/schema/buyer-copy change.
**scope:** generate a small fresh batch after Evidence-Sufficiency Band Gate v1 and verify buyer-advertised strength now behaves correctly when evidence is thin.

## Purpose

Evidence-Sufficiency Band Gate v1 added a deterministic buyer-projection gate that caps a thin-evidence read's advertised band from High to Medium while preserving the lean and the raw numeric confidence. This slice validates that gate on freshly generated artifacts.

Primary question: does the band gate improve buyer honesty without flattening useful leans or changing raw confidence? Answer: yes.

## Scope

- start/verify local services; generate a small fresh batch through the full `.NET POST /api/agent-runs` chain.
- inspect each artifact and apply the gate logic (the unit-tested `evidenceSufficiency` + `gateAdvertisedStrength` from `buyer-signal-summary.ts`).
- verify advertised-strength gating, lean preservation, raw-confidence preservation, buyer safety, and cost telemetry.

## Non-goals

- no prompt/model/cost-guardrail/source/probe-refresh change.
- no lean, raw confidence, confidence-constant, or SportsEvaluator change.
- no schema/contract or buyer-copy change.
- no pricing/Stripe/auth/tenant/dashboard/deployment change. no Jera change. no push.

## Service readiness summary

- Docker/`devcore-sql` up on 1433; FastAPI `/api/ping` 200; .NET `/health` 200. All healthy before generation.

## Sample set generated

Six fresh artifacts via the calibration harness (full `.NET` chain), saved under the accepted convention `04 Products/sports-v1/calibration/`:
- 5 MLB (2026-06-09 single-day slate; all single-signal `starting_pitching` runs).
- 1 NBA (Spurs at Knicks, 2026-06-10; the only upcoming NBA game).

6 model calls total (one `gpt-4o-mini` per run).

Limitations (not faked): MLB grounds only `starting_pitching` today (`maxGroundedSignals == 1`), so every MLB run is `evidenceRichness 1` (thin) -- which is exactly the gate's target case, reproduced 5/5. No priors-only (`evidenceRichness 0`) and no rich (`>= 3`) fresh case appeared; NBA gave one moderate (`evidenceRichness 2`) case. The thin->High->capped path and a moderate->not-capped control are both covered; the priors-only and rich branches remain covered by the Evidence-Sufficiency Band Gate v1 unit tests, not by fresh data this batch.

## Generation method

`run-artifact-calibration.ps1 -Competition mlb -Days 3 -Take 5 -Force` and `-Competition nba -Days 14 -Take 1 -Force`. Each run: `.NET /api/agent-runs` -> FastAPI `/api/sports/analyze` (one model call) -> evaluate/quality_check/compose -> SQL persist -> artifact fetched. The band gate itself is the Angular buyer projection; it was applied to each fetched artifact's `confidence` + `evidenceRichness` using the exact gate logic to validate the buyer-advertised outcome.

## Artifact table

All six: `artifactVersion sports_decision_artifact_v3`, `cognitiveProtocol` present, `posture monitor`, buyer projection present, persisted.

- MLB Mariners at Orioles (b96b433e): raw confidence 0.75, rawBand High, evidenceRichness 1 (thin) -> advertised Medium, capped, reason recorded. Lean "Slight lean toward Orioles based on home field advantage."
- MLB Diamondbacks at Marlins (bb6b433e): 0.75, High, 1 (thin) -> Medium, capped. Lean "Slight lean toward Diamondbacks based on starting pitching advantage."
- MLB Red Sox at Rays (be6b433e): 0.75, High, 1 (thin) -> Medium, capped. Lean "Slight lean toward Rays based on home field advantage and starting pitching matchup."
- MLB Yankees at Guardians (c46b433e): 0.75, High, 1 (thin) -> Medium, capped. Lean "Slight lean toward Yankees based on starting pitching advantage with Gerrit Cole."
- MLB Twins at Tigers (c66b433e): 0.75, High, 1 (thin) -> Medium, capped. Lean "Slight lean toward Twins based on starting pitching matchup."
- NBA Spurs at Knicks (ca6b433e): raw confidence 0.675, rawBand Medium, evidenceRichness 2 (moderate) -> advertised Medium, not capped, no reason. Lean "Slight lean toward New York Knicks based on market favoring them by 1.5 points."

## Band gate behavior findings

Correct on every fresh artifact.
- thin evidence + raw High -> advertised Medium on all 5 MLB runs; `advertised_strength_limited_by_evidence` recorded internally each time.
- the lean is preserved on every capped run ("Slight lean toward ..." stays intact; the gate caps strength, not direction).
- raw numeric confidence is preserved (0.75 MLB / 0.675 NBA remain the persisted values; the gate changes presentation only).
- moderate evidence is not capped (NBA evidenceRichness 2 -> Medium unchanged); the NBA run also confirms the gate never touches a band that was not High.
- no thin+Medium or thin+Low fresh case occurred (all MLB raw confidence was 0.75 = High), and no priors-only/rich case occurred; those branches stay covered by unit tests.
- buyer copy does not expose the internal reason: a buyer-field scan that explicitly searched for `advertised_strength_limited_by_evidence` and `evidenceRichness` found 0 hits.

## Buyer credibility findings

Pass. The gate improves honesty without flattening usefulness.
- direction is still told plainly ("Slight lean toward X based on Y") on every artifact -- the lean survives the cap.
- the artifact no longer over-advertises: a single-signal MLB read now presents as Medium strength, matching its thin evidence, while still giving a usable direction, counter-case, and watch item.
- the contrast is now coherent: a "slight lean" with Medium advertised strength reads as measured decision support, not a strong call. The NBA moderate case stays Medium with two grounded signals, which is internally consistent.
- buyer language stays plain and credible; no internal/debug/factory terms in buyer copy.

## Repetition findings

Controlled, consistent with prior validations; not buyer-trust-damaging. "Slight lean toward" appears 6/6 (the lean template, by design). MLB bases split between "starting pitching advantage/matchup" (4, with specifics like "Gerrit Cole") and "home field advantage" (2). Specific detail keeps the lines from collapsing into one phrase. No betting-adjacent or trust-damaging repetition. No phrase-bank work warranted from this batch.

## Safety / toutiness findings

Pass. Buyer-visible field scan (lean, summary, counterCase, watchFor, whatWouldChangeTheRead) across all 6 returned 0 unsafe hits, including 0 hits for the internal humility-reason string and `evidenceRichness`. No lock/guarantee/hammer/free-money/unit/aggressive language; all postures `monitor`; no internal guardrail jargon leaked to buyer copy.

## Cost telemetry findings

Intact and unchanged (Artifact Cost Guardrails v1 untouched). 6 cost records emitted (one per run), all `gpt-4o-mini`, `finishReason stop`, `status ok`. Tokens 3058-3448; estimated cost $0.0006414-$0.0007089 per run (batch ~$0.004); latency 5.7-11.5s. One model call per run; zero new model-call sites (this slice changed no code); no cost-guardrail change.

## Skill update review

No skill changed. The named custom DAI skills (dai-grill-with-vault, dai-token-tight, dai-agent-handoff) and jera-workspace-skills are not present in this workspace; only `dai/.claude/skills/dai-signal-follow-up-diagnostics` exists (different domain). Recommended future skill capture (documented, not applied): a durable DAI doctrine principle -- "gate buyer-presentation humility at the deterministic projection layer; preserve raw internal calibration signals (confidence/lean) for the outcome loop; keep confidence separate from evidence sufficiency; validate doctrine changes with fresh artifacts before per-sport scaling." This validation confirms the principle held under fresh generation; it should seed a DAI doctrine/handoff skill if/when one is created.

## Pass/fail judgment

**Pass.** The band gate works on fresh artifacts: thin evidence + raw High is capped to advertised Medium with an internal humility reason, the lean and raw numeric confidence are preserved, moderate evidence is not capped, buyer copy stays clean and safe, and cost telemetry is intact. No immediate fix needed. The only caveat is sample diversity (no priors-only/rich fresh case this batch due to slate/source limits), which the unit tests cover.

## Recommended next slice

- Outcome Reconciliation v1 -- the missing feedback loop. The raw numeric confidence (deliberately preserved here) and the provisional thin/moderate/rich thresholds can only be validated/recalibrated against real game-day outcomes (deferred-ledger entry 12). This is the natural next step now that the humility gate is proven.
- Secondary: a source/grounding expansion for MLB (it grounds only `starting_pitching` today, so every MLB read is structurally thin); but that is source work, deliberately out of scope here and gated by provider availability (ledger entry 19).

## What was not changed

- no prompt, model call, parser, or cost guardrail
- no .NET SportsEvaluator, composer, constants, or numeric confidence
- no raw confidence, lean, posture, or decision logic
- no artifact schema / contract
- no Angular code (the gate from Evidence-Sufficiency Band Gate v1 was validated, not modified)
- no source/probe-refresh/pricing/Stripe/auth/tenant/dashboard/deployment change
- no Jera change
- no `OPENAI_API_KEY` or secret change

## Validation performed

- service health: FastAPI ping 200; .NET health 200; Docker/devcore-sql up.
- 6 runs completed and persisted; all v3 + cognitiveProtocol; buyer projection present.
- gate applied to each fresh artifact: 5/5 MLB thin+High -> Medium capped with reason; NBA moderate -> not capped.
- raw confidence + lean preserved on every artifact.
- buyer-field safety scan: 0 unsafe hits, 0 internal-reason leaks.
- cost telemetry: 6 records, intact.
- docs/artifacts checks: `git diff --check`, added-line exact-path scan, vault non-ASCII added-line scan.
