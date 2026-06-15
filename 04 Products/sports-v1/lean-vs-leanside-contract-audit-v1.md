# Lean vs LeanSide Contract Audit v1

**date:** 2026-06-15
**status:** audit only. no code change, no model spend, no outcomes/evaluations written. conclusion: the two fields are intentionally distinct; `LeanSide` is justified and not redundant.
**classification:** contract verification (no cleanup required).

## Question

Do `lean` and `LeanSide` represent the same concept (so the system should reuse one), or are they intentionally separate?

## Answer

Intentionally separate, with non-overlapping value domains and non-overlapping consumers. `lean` is free-text buyer-facing prose; `LeanSide` is the normalized machine-readable side the deterministic evaluator/matcher require. `lean` cannot safely substitute for `LeanSide` without reintroducing display-name parsing -- the exact fragility Outcome Reconciliation Contract v1 forbids. **Keep `LeanSide`.**

## Field map

| field | layer(s) | type | value domain | source | persisted? | consumers | purpose |
|---|---|---|---|---|---|---|---|
| `lean` / `Lean` | FastAPI response -> artifact OutputJson -> `AgentRunArtifactDto` -> Angular | string (free text) | one sentence naming the favored side by **actual team name** + the single strongest reason (e.g. "Slight lean toward Chiefs based on rest advantage"); "No clear lean" / "Mixed read" when none. Explicitly forbidden from using "home"/"away" as the subject, and from betting-posture wording. | model (`lean`), sanitized in `_parse_response` | yes (in OutputJson) | Angular `analyzer.component.ts:180` (`r.lean`, display); artifact inspection DTO | buyer-facing human-readable directional read + reason |
| `lean_side` / `LeanSide` | FastAPI response -> `SportsComposer` -> `AgentRun.LeanSide` column + OutputJson -> `AgentRunArtifactDto` | string? | exactly `home` \| `away` \| `null` | model (`lean_side`), clamped to home/away/null in `_parse_response` (`sports_analyzer.py:419-420`); passed through `SportsComposer.cs:47`; denormalized `AgentRunsController.cs:102` | yes (denormalized column **and** OutputJson) | `RunEvaluator.Evaluate` / `OutcomeReconciliationMatcher` (evaluation/reconciliation); artifact DTO since `1d9ae93` | normalized machine-readable side for deterministic correctness grading |

## Source-of-truth finding

- **Directional evaluation / reconciliation:** `LeanSide` is authoritative. `RunEvaluator.Evaluate(leanSide, outcomeStatus)` compares `leanSide` (home/away) against `WinningSide(outcomeStatus)` (home/away from home_win/away_win). It requires the normalized trio; prose `lean` is never read by the evaluator or matcher.
- **Display:** `lean` is authoritative for the buyer-facing read; Angular renders `r.lean` and never touches `leanSide`.
- The two are **coupled** by a prompt consistency rule ("`lean_side` must agree with `lean`": `sports_analyzer.py:128-129`) but represent different concerns. One source of truth per concern -- no conflict, no two-sources-of-truth-for-one-concept problem.

## Provenance (what existed first)

- `lean` (prose) is the original analyzer output field; it predates the structured side.
- `lean_side` (FastAPI) and `LeanSide` (.NET) were introduced together specifically to make outcome evaluation possible: `43f9385` "feat(sports-v1): add structured lean_side to analyze response **and activate run evaluation**" and `2af54c7` "feat(sports-v1): add deterministic run evaluation for resolved outcomes". So `LeanSide` was purpose-built for the evaluator, not added as a redundant mirror of `lean`.
- The prior `1d9ae93` (Directional Lean Contract v1) did **not** add a parallel field -- it projected the already-persisted `LeanSide` onto the read DTO (`lean` was already projected). So there was never an "added LeanSide because lean projection was missing" situation; both the persisted structured side and the prose already existed.

## Consumer map

- `lean` -> Angular buyer display (`analyzer.component.ts:180`); artifact inspection DTO.
- `LeanSide` -> `RunEvaluator`, `OutcomeReconciliationMatcher`/`OutcomeReconciliationService`, `AgentRun.LeanSide` query column, artifact inspection DTO (read-only, since the prior slice). No buyer surface consumes `LeanSide`.

No third structured side field exists; no other source derives a competing directional value.

## Answers to the audit questions

1. **What existed first?** `lean` (prose). `lean_side`/`LeanSide` came later to activate evaluation.
2. **Type/domain of `lean`?** free-text string; a sentence with the team name and reason.
3. **Structured or free text?** free text (display prose).
4. **Does `lean` encode home/away/null?** No -- it names the **team** (e.g. "Chiefs"), not home/away; null case is the phrase "No clear lean".
5. **Does `lean` include posture/recommendation?** No -- posture is a separate field; `lean` is explicitly barred from betting-posture wording.
6. **Does `lean` vary by sport/market?** Same contract shape across sports; only the team names differ.
7. **Is `LeanSide` persisted or only DTO?** Persisted: denormalized `AgentRun.LeanSide` column **and** inside OutputJson; also projected on the DTO.
8. **Is `LeanSide` derived from artifact/model/lean?** From the **model** (`lean_side`), passed through unchanged. Not parsed from prose `lean`.
9. **Does reconciliation require home/away/null?** Yes -- the evaluator compares against `WinningSide` (home/away).
10. **Could the evaluator use `lean` instead?** No -- it would have to parse a team name from prose and resolve home/away, reintroducing display-name fragility the reconciliation contract forbids.
11. **Two sources of truth for one concept?** No -- two representations for two concerns (display reason vs structured direction).
12. **Which is authoritative?** `LeanSide` for evaluation; `lean` for display.
13. **Do the new tests assert the right contract?** Yes -- they assert the structured side surfaces (home) and that a true null passes through; they prove the distinct field is projected, they do not lock in duplication.

## Duplication verdict

No duplication. `LeanSide` carries a distinct, stable meaning (`home`/`away`/`null`) that the existing `lean` field cannot safely provide. The decision-rule branch that applies is "existing `lean` is display-oriented / textual / posture-free but team-named -> keep `LeanSide` as the normalized machine-readable evaluation key; keep `lean` for display." Met.

## Recommendation

**Keep `LeanSide`.** No rename, no deprecation, no cleanup. No code change this slice. Optional future hardening (not required, not done): a lightweight invariant check that `lean_side` and `lean` agree (e.g. flag when `lean_side` is null but `lean` names a side, or vice versa) -- a consistency guard, not a structural change. Logged as a watch item, not a slice.

## Risks / compatibility

- None introduced (audit only). The fields stay as-is; existing artifacts, DTO consumers, and the matcher are unaffected.
- Standing consistency dependency: `lean` and `LeanSide` must remain in agreement; today this is enforced only by prompt instruction + the deterministic clamp, not by a runtime assertion. Low risk (the evaluator only reads `LeanSide`, so a divergence would affect display wording, not correctness), but worth the optional guard above if divergence is ever observed.

## Recommended next slice

Unchanged primary action: **reconcile the 7 directionally usable MLB runs after settlement** through the proven `POST /api/agent-runs/reconcile` statsapi path. The optional `lean`/`LeanSide` consistency guard can ride along only if a divergence is actually observed in generated artifacts. WNBA remains deferred; priors-only orientation remains a deferred operator doctrine decision.
