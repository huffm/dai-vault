---
title: "Backed-Depth Divergence Cohort Integrity QA 2026-07-06 v1 -- PASSED"
type: "report"
date: "2026-07-06"
status: "COMPLETE -- QA passed; 6-run cohort is settlement-ready; no blockers"
project: "DAI"
slice: "Backed-Depth Divergence Cohort Integrity QA v1"
repos:
  dai: "unchanged (d79c38f)"
  dai-vault: "docs-only"
tags:
  - calibration
  - cohort
  - backed-depth
  - divergence
  - qa
related:
  - "06 Execution/reports/backed-depth-divergence-capture-2026-07-06-v2.md"
  - "06 Execution/reports/backed-depth-divergence-candidate-slate-2026-07-06-v2.md"
  - "06 Execution/reports/backed-depth-divergence-cohort-integrity-qa-handoff-2026-07-06-v1.md"
---

# Backed-Depth Divergence Cohort Integrity QA 2026-07-06 v1 -- PASSED

## 1. objective

No-spend, read-only pre-settlement QA of the 6-run 2026-07-06 backed_depth divergence
cohort. Verify every run is complete, identity-safe, market-baselined, registry-routed,
cost-accounted, and settlement-ready -- WITHOUT reconciling, generating, or changing code.

## 2. repo state

- dai: `d79c38f`, 0 ahead / 0 behind, dirty only on pre-existing `DevCore.Data.csproj`
  phantom (empty textual diff).
- dai-vault: `aa8c82f`, 2 ahead / 0 behind (unpushed: capture v2 + readiness), untracked
  `06 Execution/system-state-synopsis-v1.md`.
- Services: DevCore.Api :5007 = 200; devcore-sql Up; agent-service :8000 DOWN (no
  generation). Read-only DB + read-only endpoints only.

## 3. cohort membership (Phase 1)

Exactly 6 target runs found (one per gamePk), no duplicates (total_runs = active_runs = 1
each), none superseded, all `sports.matchup.analysis` / mlb / season 2026, all `completed`,
all `ExclusionReason = NULL`, all `SourceProvider = mlb_statsapi`, all with `ExternalGameId`
= StatsAPI gamePk. Completed 2026-07-06 12:41-12:43 UTC, durations 4.7-8.1s.

| gamePk | AgentRunId | matchup (away @ home) | LeanSide | Status | ExclusionReason | Superseded |
|---|---|---|---|---|---|---|
| 822958 | AC31433E-F36B-1410-8175-00373DB4B724 | new-york-yankees @ tampa-bay-rays | home | completed | NULL | NULL |
| 822712 | AD31433E-F36B-1410-8175-00373DB4B724 | houston-astros @ washington-nationals | home | completed | NULL | NULL |
| 824900 | B331433E-F36B-1410-8175-00373DB4B724 | new-york-mets @ atlanta-braves | home | completed | NULL | NULL |
| 823036 | B431433E-F36B-1410-8175-00373DB4B724 | milwaukee-brewers @ st-louis-cardinals | home | completed | NULL | NULL |
| 823282 | B631433E-F36B-1410-8175-00373DB4B724 | arizona-diamondbacks @ san-diego-padres | home | completed | NULL | NULL |
| 823205 | B731433E-F36B-1410-8175-00373DB4B724 | toronto-blue-jays @ san-francisco-giants | away | completed | NULL | NULL |

## 4. registry / backed_depth provenance (Phase 2)

Read from the durable `AgentRuns.PromptRouteProvenanceJson` column (not just the in-memory
artifact). All 6 identical on the route-critical fields:

- `promptSource = registry`
- `legacyFallbackUsed = false`, `fallbackReason = null`
- `regimeAllowlisted = true`
- `selectedDataRegime = starter_enriched_market_backed_depth`
- `selectedPromptRecipeId = mlb.pregame.analysis.starter_enriched_market_backed_depth.v1`
- `attributionStatus = complete`
- `assembledHash` length = 64 (per-game distinct, e.g. 823036 = 9ed4960d..., 822958 = a59a20b4...)

Source depth (persisted `OutputJson.SourceDepth`) on all 6: starting_pitching `enriched`,
market_odds `enriched` (backed_depth), bullpen_availability `shallow`. `EvidenceRichness = 2`
and `Publishability = Publishable` on all 6. No run fell back; none is off-regime -> no
provenance blocker.

## 5. market baseline verification (Phase 3)

Market baseline is persisted at decision time. It lives in two places:
`OutputJson.SignalAvailability` (a grounded `market` entry: `Source=odds_api`,
`Quality=usable`, `DecisionUse=directional_only_without_confirmation`,
`ConfidenceEffect=support_cautiously`) and `OutputJson.SourceDepth.market_odds.Detail` (the
numeric baseline string: book count, consensus side, median home win prob, book
disagreement). All read from persisted decision-time data -- nothing recomputed from
external sources.

| gamePk | DAI lean | conf | consensus side | median home win P | book disagr | books | LeanAgreement | DAI-vs-market |
|---|---|---|---|---|---|---|---|---|
| 822958 | home | 0.75 | home | 53% | 3% | 9 | Consistent | AGREE |
| 822712 | home | 0.75 | home | 54% | 1% | 9 | Consistent | AGREE |
| 824900 | home | 0.80 | home | 57% | 1% | 9 | Consistent | AGREE |
| 823036 | home | 0.75 | **away** | 50% | 4% | 9 | Consistent | **DISAGREE** |
| 823282 | home | 0.75 | home | 53% | 2% | 9 | Consistent | AGREE |
| 823205 | away | 0.75 | away | 51% | 2% | 9 | Consistent | AGREE |

Known agreement summary confirmed: **5 agree, 1 disagree**; the disagreement is 823036
Brewers @ Cardinals. `LeanAgreement.Status = Consistent` on all 6 (this is the artifact's
prose-vs-structured lean integrity check, not the market flag).

Note on flag shape: there is no discrete stored "DAI-market agreement" boolean or structured
"market favorite / implied prob / de-vig gap" object. DAI-vs-market agreement is DERIVED
from `LeanSide` vs the consensus side in the market_odds Detail. This is sufficient for
settlement (which matches by gamePk) but is a semi-structured baseline -- see Risks.

## 6. cost / model verification (Phase 4)

- 6 model-call cost lines, all `model = gpt-4o-mini`, one per run.
- per-run estimated cost $0.000699-$0.000735; **total $0.004259** (re-summed this slice from
  the captured stdout). Cap $0.05 never approached (8.5%).
- Limitation: cost lines were emitted to the `devcore.cost` logger on agent-service stdout,
  captured to a session-scoped scratchpad log -- NOT a durable file/DB sink. The durable
  aggregate is preserved in the committed capture report v2. If per-run cost auditing must
  survive the session, a persistent cost sink would be needed (out of scope; no code change).

## 7. settlement readiness (Phase 5)

- Outcome/evaluation rows referencing the 6 runs = **0 / 0** (join on AgentRunKey). Cohort is
  entirely unsettled, as intended.
- `GET /api/agent-runs/reconcile-precheck` (read-only, advisory, writes nothing) for each of
  the 6 gamePks returns `activeRunCount = 1`, `unreconciledActiveCount = 1`,
  recommendation **SingleMatch -- "Identity POST /reconcile is safe."** No identity
  collision, no ambiguity.
- Provider identity stable (`mlb_statsapi` + gamePk), `SourceProvider` present,
  `ExternalGameId` present, no duplicate identity. Settlement via identity `/reconcile` is
  expected to be clean for all 6 once finals exist.

## 8. divergence run focus (823036 Brewers @ Cardinals, b431433e)

- AgentRunId `B431433E-F36B-1410-8175-00373DB4B724`; identity `mlb_statsapi:823036`;
  completed; `ExclusionReason = NULL`; SingleMatch reconcile-safe.
- Provenance: registry, no fallback, backed_depth, attribution complete, hash 9ed4960d...
- Decision: DAI leaned **home St. Louis Cardinals** at confidence 0.75; market consensus
  **away Milwaukee Brewers** (median home win prob 50%, book disagreement 4%, 9 books) ->
  the only DAI-vs-market disagreement, and the first captured backed_depth divergence.
- `LeanAgreement.Status = Consistent` (prose supports the structured home lean). Evidence
  richness 2, posture monitor, publishable.
- This run is the highest-information member for settlement: it is the one place the cohort
  can produce edge-over-market signal (if DAI's Cardinals win while the market's Brewers
  were favored).

## 9. risks / blockers

- **No blockers to settlement.** All 6 are identity-safe, unsettled, SingleMatch, registry
  backed_depth.
- Risk (minor): cost evidence is non-durable (session stdout only); durable aggregate lives
  in the capture report, not a DB/file sink.
- Note (minor): market baseline is semi-structured (numeric detail in a string; agreement
  derived, not stored). Settlement does not depend on it (matches by gamePk), but any future
  automated edge computation should parse the Detail or add a structured market snapshot.
- Reminder: finals are not yet available (all 6 pre-game at QA time, first pitch 14:10 ET);
  settlement must wait for official finals.

## 10. calibration impact projection (Phase 6)

Without writing outcomes:

- If DAI wins the divergence run (823036, Cardinals) and the market's favorite (Brewers)
  loses, readable disagreement-with-correct-direction evidence improves -- the first such
  data point.
- If DAI loses 823036 and the market is right, the existing `discrimination_inverted` /
  disagreement concern worsens (market looked better on the one game we diverged).
- If the 5 agreement games dominate the settled signal, the agreement sample grows but
  edge-over-market evidence stays limited (agreement games cannot demonstrate independent
  edge).
- Gate 4 likely stays FALSE regardless: one divergence data point does not materially move
  `insufficient_market_disagreement` / `insufficient_market_coverage`, and confidence is
  nearly flat (0.75 x5, 0.80 x1) so discrimination is unlikely to separate. No outcome claim
  before final scores.

## 11. validation proof

- Cohort: 6/6 target runs present, 0 duplicates, 0 superseded, all completed + active +
  ExclusionReason NULL.
- Provenance (durable column): 6/6 registry, fallback=false, regime backed_depth, attribution
  complete, 64-char hash.
- Market: 6/6 baselined (9 books each); derived agreement 5 agree / 1 disagree matches the
  known summary.
- Cost: 6 gpt-4o-mini lines, total $0.004259, one per run.
- Settlement: 0 outcomes + 0 evaluations for the cohort; 6/6 SingleMatch reconcile-safe.
- Read-only: no `/reconcile`, no generation, no model call, no writes issued this slice.

## 12. what did not change

No runtime code, prompts, prompt registry recipes, routing, confidence generation,
calibration gate, buyer copy, schema/migrations. No AgentRuns / outcomes / evaluations
written (still 279 / 112 / 112). agent-service not started. `.env` untouched. dai HEAD
`d79c38f` unchanged (only csproj phantom).

## 13. recommended next slice

**Backed-Depth Divergence Settlement / Reconciliation v1** -- after 2026-07-06 official
finals, settle the 6 runs via identity `/reconcile` (all SingleMatch), write outcomes +
evaluations, then re-read pooled calibration. Watch 823036 (the divergence). Settlement-only;
no generation, no gate tuning.
