# Discern Improvement Review v1

**date:** 2026-06-12
**status:** review, docs only. no runtime, test, prompt, schema, or buyer change. produced during the wait window before Stage 0 calibration candidates settle.
**scope:** the Discern macro stage (Weigh, Contrast, Stress) of the Cognitive Protocol Runtime. assesses current state and whether Discern needs improvement now or after reconciliation evidence exists.

## 1. Executive summary

Discern is the most code-heavy and best-doctrined of the four reasoning stages, but its *production* surface is narrow and split across three layers: model-emitted prose (`DiscernProtocol.Weigh/Contrast/Stress`), a rich platform-owned deterministic surface that doctrine assigns to Discern but that currently lives under Interrogate/Perceive plumbing (`SignalQualityEvaluator`, `SignalFollowUpEvaluator`, the fallback ladder), and one buyer-visible field (`watch_for` <- `discern.stress`). Two dormant Discern runners exist (`DiscernStationRunner`, `ProbeRefreshDiscernReweigh`) and neither has a production caller.

Recommendation: **do not improve Discern code now.** The single highest-value, lowest-risk improvement is documentation -- closing the doctrine-vs-code naming gap (where the canonical Discern micro-actions actually live today) so the next builder does not mistake the dormant `DiscernStationRunner` for the real Discern surface. Everything that would *activate* Discern (wiring the runner, moving Stress out of `interrogate.stress`, deterministic weigh/contrast helpers) should wait until reconciled-outcome evidence exists, because Discern's job is to grade evidence weight and that grading cannot be validated without outcomes. This review progresses deferred ledger entry 16; it resolves nothing.

## 2. Current Discern state

Four distinct things all called "Discern" today:

1. **Implemented production behavior (model-emitted prose).** `DiscernProtocol(Weigh, Contrast, Stress)` -- three nullable strings, defined in `CognitiveProtocol.cs`. The model emits them in the analyzer `discern` block; the platform passes them through unchanged via `CognitiveProtocolBuilder`, projects them via `ProtocolVocabularyMapper` / `CognitiveProtocolView` into `DiscernProtocolView`. The protocol registry marks Discern stations `ExecutionOwner.Model`. The platform performs no Discern reasoning on these fields -- it carries them.

2. **Implemented production behavior (deterministic surface doctrine assigns to Discern).** Per `phases/discern.md`, the real structured judgment of Discern already exists and runs every sports run: `SignalQualityEvaluator` (per-signal quality / decision-use / confidence-effect), `SignalFollowUpEvaluator` (follow-up resolution + the six-tier sharp/public fallback ladder), and `SportsEvaluator` (grounded-count -> confidence band). This is the richest deterministic Discern surface in the platform, but in code it sits inside `SportsRetrievalOutput` / signal plumbing, not under a "Discern" namespace.

3. **Dormant scaffolding.** `DiscernStationRunner` / `IDiscernStationRunner` (generic, platform-owned, shape-only) and `ProbeRefreshDiscernReweigh` (probe-refresh-specific, signal-keyed). Neither is DI-registered, wired into `ProtocolNodeRunner`, or consumed by any production path. Tracked as ledger entry 16.

4. **Docs-only doctrine.** `phases/discern.md`, `protocol-vocabulary-map.md`, `signal-fallback-ladder.md` describe the canonical Discern model (Weigh = grade signals, Contrast = interpret divergence, Stress = name fragility) and the legacy-to-canonical field mapping.

**Artifact fields that approximate Discern:** `DiscernProtocol.Weigh/Contrast/Stress` (model narrative), `signalAvailability` quality/decision-use/confidence-effect, `signalFollowUps` resolution + fallback type/equivalence, and the buyer `watch_for` field. **Buyer-visible:** `watch_for` (rendered as "What Could Change the Read") is sourced directly from `protocol.discern.stress` in `sports_analyzer.py`.

**Tests protecting Discern:** `DiscernStationRunnerTests` (dormant runner shape), `ProbeRefreshDiscernReweighTests` (dormant reweigh), `ProtocolRegistryTests` (Discern station cards + `DiscernContrast` model ownership), `ProtocolToolAccessPolicyTests` (tool access at `DiscernWeigh`), `ProtocolVocabularyMapperTests` and `SportsComposerTests` (weigh/contrast/stress pass-through). The deterministic Discern surface is protected by the signal evaluator tests, not by anything named "Discern".

## 3. Weigh / Contrast / Stress review

**Weigh -- how signals are weighted or prioritized.** Two halves. Deterministic: `SignalQualityEvaluator` grades each signal `strong / usable / unavailable`, assigns decision use (`primary_signal / confirmation / proxy_candidate / directional_only`) and a confidence effect (`support / support_cautiously / dampen / block_aggressive_posture / neutral`); `SportsEvaluator` maps grounded count + confidence-effect flags to a confidence band. Qualitative: the model's `discern.weigh` sentence names "what is admitted, what is missing, what should be discounted." This is the strongest of the three micro-actions today and is real, deterministic, and platform-owned.

**Contrast -- how opposing signals are compared.** Weakest in production. The model's `discern.contrast` sentence interprets market / sharp-public / external signals when present (or null if no market data). There is no deterministic contrast engine that detects two grounded signals disagreeing and quantifies the divergence; contrast is model narrative only. The dormant `ProbeRefreshDiscernReweigh` is the closest thing to structured contrast (it classifies a refreshed signal as SupportsRead / WeakensRead / Neutral / NeedsHumanReview against an existing read) but it is dormant and single-signal, not a cross-signal comparator.

**Stress -- how fragility or evidence weakness is tested.** Single-source by canonical contract (Stress Collapse v1 retired the legacy `interrogate.stress` / `discern.test` duplication). The model's `discern.stress` sentence names "the fragility, key risk, or condition under which the read fails" and is the one Discern micro-action that surfaces to the buyer verbatim (`watch_for` / "What Could Change the Read"). Deterministically, `block_aggressive_posture` propagation and the evidence-sufficiency band gate act as a structural stress check on over-confidence, but the narrative stress is model-authored and unvalidated against outcomes.

## 4. Implemented vs dormant vs docs-only

| layer | what | status |
|---|---|---|
| `DiscernProtocol.Weigh/Contrast/Stress` | model-emitted prose, passed through | implemented, live every run |
| `SignalQualityEvaluator` / `SignalFollowUpEvaluator` / fallback ladder | deterministic evidence grading (doctrinal Discern.Weigh) | implemented, live, but namespaced under signal plumbing not "Discern" |
| `SportsEvaluator` band | grounded-count -> confidence band | implemented, live |
| `watch_for` <- `discern.stress` | buyer "What Could Change the Read" | implemented, live, buyer-visible |
| `DiscernStationRunner` / `IDiscernStationRunner` | generic platform runner, shape/input validation only, no weighing | dormant, no caller (ledger 16) |
| `ProbeRefreshDiscernReweigh` | signal-keyed reweigh with effect classification | dormant, probe-refresh-specific (ledger 14, 16) |
| `phases/discern.md`, vocabulary map, fallback ladder | canonical Discern doctrine + legacy mapping | docs-only doctrine |

The central fact: **the canonical Discern micro-actions and the code do not line up by name.** Doctrine puts Stress under Discern; code emits it under `interrogate.stress` and the prompt's `discern` block. The deterministic Discern surface lives under signal plumbing. The thing literally named `DiscernStationRunner` does the least actual discerning (it validates input shape and returns "input available for assessment"). A new builder reading only the code would misread where Discern lives.

## 5. Gaps and risks

- **Naming/locus gap (doctrine risk, highest).** The real Discern surface is not where its name is. Without a clear map, a future slice could "activate Discern" by wiring `DiscernStationRunner` and add a shape-validation no-op while believing it added reasoning.
- **Contrast is narrative-only (capability gap).** No deterministic detection of grounded-vs-grounded signal divergence. Acceptable today (sharp_public, the main contrast input, is the dominant *missing* signal per Buyer Artifact UX Calibration v1), but it means Contrast cannot be calibrated.
- **Stress is buyer-visible but unvalidated (trust risk).** `watch_for` ships to buyers straight from a model sentence with no outcome feedback. Honest framing currently mitigates it, but its quality is unmeasured until reconciliation exists.
- **Two dormant Discern runners with overlapping intent (entropy risk).** `DiscernStationRunner` (generic) and `ProbeRefreshDiscernReweigh` (specific) both claim Discern assessment; a second consumer could copy the wrong one. Tracked by ledger 14 and 16.
- **No Discern calibration loop yet.** Weigh produces a confidence band, but no reconciled outcome has ever flowed back to test whether the weighting was right (ledger 12, 25).

## 6. Improvement opportunities (ranked)

1. **Doctrine clarity (do now, docs only).** This review + the entry-16 progression *is* the improvement: a single authoritative map of where each Discern micro-action lives in code today vs doctrine, and an explicit "the dormant runner is not the live Discern surface" note. Lowest risk, prevents the most expensive future mistake. Already reflected in `phases/discern.md` legacy mapping; this review consolidates it for the buyer-product context.
2. **Discern internal assessment schema (design only, defer build).** Specify the shape Discern *would* pass to Decide (ranked signal weights, detected conflicts, fragility notes, unresolved residue, posture constraints) as a contract, before any runner is wired. Cheap to write, no activation.
3. **Discern calibration-feedback plan (design only, defer build).** Define how reconciled outcomes will test Weigh's confidence band and Stress's fragility calls, so the calibration surface (ledger 12/25) has a Discern-shaped target when evidence exists.
4. **Characterization tests for the live deterministic Discern surface (small, optional).** Lock current `SignalQualityEvaluator` -> band behavior as Discern characterization, so a future doctrinal rename does not silently change grading. Only if a rename slice is actually scheduled.
5. **Deterministic Contrast helper (defer).** A pure function detecting grounded-vs-grounded divergence. Real capability gain but speculative until contrast inputs (sharp_public, market) are reliably grounded, which they are not.

## 7. What not to build yet

Explicitly defer all of:

- **activating `DiscernStationRunner`** -- no production caller exists (ledger 16, 17, 18); activation would add a shape-validation no-op mistaken for reasoning.
- **adding model calls / expanding the analyzer** -- the single model-call boundary is metered and bounded; Discern needs no new call.
- **moving Stress out of `interrogate.stress`** into a canonical `discern.stress` runtime slot -- a real future code slice, but it touches the analyzer prompt, parser, persisted shape, and buyer `watch_for` source; out of scope and not urgent.
- **new agents, dashboards, or a calibration read surface** -- Internal Calibration Read Surface v1 stays not-ready until real evaluations exist (ledger 25 latest update).
- **changing buyer copy** ("What Could Change the Read" is locked and honest).
- **changing evaluator logic** (`SignalQualityEvaluator` / `SignalFollowUpEvaluator` / `SportsEvaluator`) or any confidence / posture / lean behavior.
- **generalizing `ProbeRefreshDiscernReweigh`** into platform Discern logic -- single consumer, would be a speculative one-use abstraction (ledger 14).
- **a deterministic Contrast engine** before contrast input signals are reliably grounded.

## 8. Recommended next Discern slice, if any

If a Discern slice is wanted, the only safe one now is **Discern Locus and Contract Clarification v1** -- docs/contract only: (a) a one-page map of where Weigh/Contrast/Stress live in code vs doctrine, (b) the Discern -> Decide internal assessment schema as a written contract (not wired), (c) the Discern calibration-feedback plan. No runner activation, no model call, no prompt change, no test change beyond optional characterization.

No code Discern slice should run before reconciled-outcome evidence exists.

## 9. Implement now or wait for reconciliation evidence

**Wait.** Discern's core function is grading how much weight evidence deserves. That grading is only validated by outcomes, and zero reconciled outcomes exist yet (ledger 25: all four Stage 0 candidates are still pre-settlement). Improving Discern *code* now would be tuning a weighting function with no error signal -- the exact intuition-over-evidence move the calibration discipline (ledger 12) forbids. The doctrine/contract clarification in section 8 is the only work that is correct to do during the wait, because it carries zero runtime risk and makes the eventual evidence-driven slice cheaper and less error-prone.

## References

- `02 Platform/architecture/cognitive-factory/phases/discern.md`
- `02 Platform/architecture/cognitive-factory/protocol-vocabulary-map.md`
- `02 Platform/architecture/cognitive-factory/signal-fallback-ladder.md`
- `02 Platform/architecture/cognitive-factory/deferred-runtime-decisions-ledger-v1.md` (entries 12, 14, 16, 17, 18, 25)
- code: `DevCore.Api/AgentRuns/CognitiveProtocol.cs`, `CognitiveProtocolBuilder.cs`, `ProtocolVocabularyMapper.cs`, `DevCore.Api/Protocols/DiscernStationRunner.cs`, `ProbeRefreshDiscernReweigh.cs`; `services/agent-service/app/services/sports_analyzer.py` (`discern` block, `watch_for`)
