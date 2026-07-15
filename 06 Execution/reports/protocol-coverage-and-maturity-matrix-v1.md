---
title: "Protocol Coverage and Maturity Matrix v1 (2026-07-15)"
type: "evidence-report"
date: "2026-07-15"
status: "complete"
project: "DAI"
slice: "DAI AI Engineering Hardening Catalog and Protocol Ready Queue v1"
repos:
  dai: "unchanged (read-only inspection at 85a8831)"
  dai-vault: "docs-only"
tags:
  - architecture
  - protocol
  - evidence
  - planning
related:
  - "02 Platform/architecture/cognitive-factory/protocol-vocabulary-map.md"
  - "06 Execution/plans/ai-engineering-hardening-catalog-v1.md"
  - "06 Execution/plans/hardening-ready-queue-v1.md"
  - "06 Execution/plans/ai-engineering-fitness-checks-v1.md"
---

# protocol coverage and maturity matrix v1

Every cell is repository-evidenced at dai `85a8831`. Maturity levels: doctrine only ->
represented in contracts -> runtime implemented -> fixture proven -> live read proven ->
paid-run proven -> settlement proven -> buyer proven -> operationally proven. Maturity is
never inferred without evidence; "buyer proven" is claimed only where a surface is
deliberately buyer-bearing.

## 0. doctrine note (recorded, NOT silently reconciled)

The operator authorization for this slice defines the canonical sequence
Perceive -> Interrogate -> Discern -> Decide -> Synthesize with 12 canonical
micro-actions = Interrogate(Question, Probe, Verify) + Discern(Weigh, Contrast, Stress)
+ Decide(Resolve, Position, Justify) + Synthesize(Integrate, Compose, Deliver), and
Perceive as the intake layer WITHOUT three doctrine-level micro-actions.

The vault's `protocol-vocabulary-map.md` (2026-05-14, "canonical runtime active")
counts the 12 as Perceive(Detect, Frame, Aim) + Interrogate + Discern + Decide, with
Synthesize explicitly "not counted among the 12." The runtime seed emits detect/frame/
aim as model fields, and the Synthesize trio is platform-completed constants.

These countings CONFLICT. This slice organizes the catalog per the operator
authorization (the newer instruction) and changes no protocol names, sequence, or
runtime shape. Reconciling the vocabulary map's counting with the authorization is
queued as ready-queue card G-01 -- an explicit operator doctrine decision, not a
silent edit.

## 1. phase and micro-action matrix

Data flow (evidence): model (gpt-4o-mini, JSON mode, temp 0.3; sports_analyzer.py:657-670)
emits the protocol seed as prose -> python null-safe parse + posture validation
(sports_analyzer.py:401-455) -> .NET SportsAnalyzerProtocolSeed -> SportsComposer.Compose
-> CognitiveProtocolBuilder.FromAnalyzerProtocolSeed (adds deterministic interrogate.probe
+ Synthesize constants) -> AgentRunExecutionResult.CognitiveProtocol -> persisted in
AgentRun.OutputJson (artifact v3) -> read-side ProtocolVocabularyMapper.Project ->
GET /api/agent-runs/{id}/artifact -> Angular /dev/artifacts (operator/dev only).

| element | computed by | validated | persisted | observable | tests | buyer exposure | maturity |
|---|---|---|---|---|---|---|---|
| Perceive intake (detect/frame/aim fields) | model-emitted prose (seed, sports.py:120-125) | null-safe only | OutputJson | /artifact ProtocolView (dev page) | parsing + 10 projection tests | none | paid-run proven (fields present across 285 v1 + 16 v2 runs) |
| Perceive intake: identity/evidence staging | deterministic (SportsRetriever; readiness classifier SourceReadiness.cs) | eligibility gate + gamePk plausibility (controller :59-63) | run row + MarketSnapshotBatch + SignalAvailability | source-readiness endpoint; /rows | DH suite 20, readiness 10, depth 13, envelope 15, fulfillment 9 | none (drives eligibility) | settlement proven (identity through 15 settlements) |
| Interrogate.Question | model-emitted prose | null-safe; counter_case falls back to it | OutputJson | ProtocolView | structural only (parse/fallback) | none | paid-run proven |
| Interrogate.Probe | DETERMINISTIC (CognitiveProtocolBuilder.BuildProbe :103-141; templates :202-209) | dedup + ordinal sort; unknown signals dropped; null when no template | OutputJson | ProtocolView + prompt-trace re-derivation (PromptTrace.cs:240,265) | 22 builder + 7 request + 6 node-execute | none | paid-run proven |
| Interrogate.Verify | model-emitted prose | null-safe only | OutputJson | ProtocolView | structural only | none | paid-run proven |
| Discern.Weigh | model prose + DETERMINISTIC surface (SignalQualityEvaluator :12-102: Quality/DecisionUse/ConfidenceEffect/FollowUpSignals) | evaluator matrix incl. sharp_public->market coupling | OutputJson + SignalAvailability | ProtocolView; availability on artifact | evaluator tests; quality-checker rule 6 | indirect only | paid-run proven (deterministic surface fixture proven) |
| Discern.Contrast | model-emitted prose | null-safe | OutputJson | ProtocolView | MarketAttributionFidelity 15 | INDIRECT: feeds market-divergence wording pick in brief (BuyerDecisionBrief.cs:156-164); strings never emitted | buyer proven (indirect, via WI-0011 live verification) |
| Discern.Stress | model-emitted prose; single-source by contract | null-safe; watch_for derives from it (:521-522) | OutputJson | ProtocolView | fallback test :483 | indirect (watchFor) | buyer proven (indirect) |
| Decide.Resolve | model-emitted prose | direction-consistency evaluator at compose + settlement 422 integrity | OutputJson | ProtocolView | consistency suites; lean-containment 11 | indirect (stance) | settlement proven (4 lean-mismatch runs caught + excluded = the control demonstrably fired) |
| Decide.Position | model-emitted, VALIDATED enum {play,pass,monitor,wait,compare,avoid} (sports_analyzer.py:401-434) | strict vocabulary; invalid -> None | OutputJson + posture deliver-extract | ProtocolView; buyer stance derives from posture | posture matrix tests ~15 | YES (stance vocabulary is buyer-bearing) | buyer proven |
| Decide.Justify | model-emitted prose | null-safe only | OutputJson | ProtocolView | structural only | none | paid-run proven |
| Synthesize.Integrate | platform CONSTANT string (CognitiveProtocolBuilder.cs:27-32) | n/a | OutputJson | SynthesizeView | mapper tests | none | represented in contracts (NOT an executed operation) |
| Synthesize.Compose | DETERMINISTIC platform op (SportsComposer.Compose :27-108; no I/O, no model) | direction-consistency evaluator inside compose | OutputJson (artifact v3 stamp) | /artifact; brief/recap exports downstream | 32 composer + 25 brief + 23 recap determinism | YES (brief/recap are its products) | buyer proven |
| Synthesize.Deliver | platform op = AgentRunResultDto mapping + persistence; label is a constant | buyer-copy safety suppression (BuyerCopySafety :76-94) | run row + OutputJson | brief/recap endpoints; ledger (procedural) | sentinel + payload tests | YES | buyer proven (test delivery); NOT operationally proven (no real buyer delivery yet -- deferred posture) |

## 2. representation vs execution findings

Independently executed (deterministic stations/operations): Interrogate.Probe,
Synthesize.Compose, the Discern.Weigh deterministic surface, the intake
identity/readiness pipeline, Decide.Position validation, direction-consistency
evaluation, buyer-copy suppression.

Model-emitted prose validated only for nullability (vocabulary exists; content never
independently evaluated): Question, Verify, Justify, Resolve (prose), Weigh (prose),
Contrast, Stress, detect/frame/aim.

Vocabulary-only / constants: Synthesize.Integrate and Synthesize.Deliver as named
operations (literal strings CognitiveProtocolBuilder.cs:27-32; the real work happens in
Compose and the controller persistence path).

DORMANT STATIONS (implemented-looking, NOT executed in production -- with doctrine
status; do NOT activate to match the diagram):

1. DiscernStationRunner (Protocols/DiscernStationRunner.cs:76-146) -- not DI-registered;
   diagnostics label it "tests only; doctrine anti-goal (do not activate)".
2. ProtocolNodeRunner.ExecuteAsync (:161-195) -- only interrogate.probe executable; all
   other 15 registry stations return UnsupportedStation; wired into no production path.
3. Probe Refresh chain (executor/assembly/merge/audit, ~18 test files) -- DI-wired
   Disabled, master switch off, no live caller; merge contract plan-only.
4. ProtocolToolAccessPolicy station-id branch -- dormant (callers pass stage sentinels).
5. CognitiveFactoryActivationOptions -- bindable config no runtime path consumes
   (Program.cs:190-195).
6. ProbeRequest structured projection -- computed + surfaced on prompt-trace; invokes
   nothing (future orchestration handoff).
7. line_movement follow-up signal -- permanently not_implemented; no probe template.

## 3. failure modes (evidenced)

- Model omits/malforms protocol -> parse degrades to None, never an error
  (test :594-603); artifact still composes; quality warnings recorded.
- Invalid posture -> None; posture falls back through deliver-extract chain.
- SportsQualityChecker (6 rules, :57-106) FAILS OPEN: warnings persisted in
  OutputJson.ArtifactQualityWarnings, deliberately NOT surfaced on any DTO/UI (:8-9) --
  the single biggest observability gap in the protocol path.
- Lean/prose contradiction -> settlement integrity 422 refuses the write (proven by 4
  excluded mismatch runs; see lean-encoding-integrity history).
- Probe: no templated gap -> null (honest absence); unknown signal silently dropped.
- ComposeFailedRun -> CognitiveProtocol null on failed runs (contract).

## 4. perceive intake checklist (OPERATIONAL GUIDANCE -- not canonical protocol vocabulary)

For each intake, the platform already executes; the operator verifies on screen:
1. identity resolved with explicit gamePk (DH-safe; never inferred from order)
2. data regime classified (starter x market; target starter_enriched_market_backed_depth)
3. grounded signal set + follow-ups recorded (SignalAvailability)
4. market snapshot linked tenant-scoped (MarketSnapshotBatch)
5. sufficiency/eligibility gate evaluated; non-eligible -> composed StopReason (fail closed)

## 5. strongest / weakest (required conclusions input)

Strongest: Decide.Position (validated enum, buyer-bearing, matrix-tested);
Interrogate.Probe (only deterministic cognitive station, 35 tests across 3 suites);
Synthesize.Compose (pure, byte-deterministic products); intake identity/readiness
(settlement proven, adversarially DH-tested); Decide.Resolve consistency enforcement
(fired in production, 422 + exclusions).

Weakest / mostly representation: Synthesize.Integrate + Deliver as named operations;
Question/Verify/Justify prose (no semantic evaluation); Perceive prose fields;
quality-warning invisibility; Angular protocol view (no spec coverage found).

Largest evidence gaps: (a) no semantic/content evaluation of prose micro-actions;
(b) no full-run replay from persisted OutputJson through projections; (c) quality
warnings unobservable; (d) no protocol-completion status on any operator surface;
(e) provider degradation states unified only at the MLB readiness layer.
