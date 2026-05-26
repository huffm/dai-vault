# Decision Factory Hardening v1

**date:** 2026-05-26
**status:** active plan. defines the internal decision artifact contract and the signal/quality doctrine the sports product manufactures against. first implementation slice (Decision Artifact Contract v1) has begun.
**scope:** how the sports decision is produced, not how it is deployed. auth, billing, tenant boundaries, Kubernetes, deployment, and account flows are explicitly out of scope here.

## 1. north star

DAI sports is a decision factory, not a pick service. Every run must manufacture a grounded, signal-aware, decision-focused artifact that a serious bettor or analyst would trust because it is honest about what it knows, what it does not, and how confident it should be. The product is worth selling when the artifact reliably shows its evidence, grades that evidence, refuses to inflate confidence past the evidence, and states the counter-case and the watch-fors. Hype is the enemy; calibrated honesty is the product.

## 2. v1 success definition

Decision Factory Hardening v1 is met when, for every supported competition, a completed run produces an artifact that deterministically surfaces:
1. which signals were available, missing, weak, or proxy-backed,
2. how each cognitive protocol phase contributed,
3. the final posture, the confidence and why it is what it is, the counter-argument, what would change the read, and where the system is uncertain,
and the platform (not the model) owns the signal-availability, signal-quality, confidence, and posture-clamp judgments. The model owns judgment text only.

Most of this contract already exists in code; v1 hardening is about naming it explicitly, pinning it with tests so it cannot silently regress, and closing the highest-value gaps (the anti-hype gate, proxy labeling, injury/availability) slice by slice.

## 3. cognitive protocol phase responsibilities

Perceive -> Interrogate -> Discern -> Decide -> Synthesize. Each phase has a fixed job and a hard boundary.

- **Perceive** (Detect, Frame, Aim): detect game context, the decision frame, available signals, missing signals, and initial uncertainty. Perceive consumes retrieval output; it does not retrieve and it must not make a pick.
- **Interrogate** (Question, Probe, Verify): question signal reliability, identify conflicts, detect stale/weak/missing signals, decide what needs follow-up. Interrogate prevents lazy reads. Probe is deterministic (`CognitiveProtocolBuilder.BuildProbe`), built from signal follow-up gaps, no model call.
- **Discern** (Weigh, Contrast, Stress): weigh evidence, establish the signal hierarchy, separate primary evidence from supporting context, surface contradictions. Discern is the anti-hype layer. The deterministic backbone is `SignalQualityEvaluator` (quality grades + confidence effects); the model narrative must agree with it, never override it.
- **Decide** (Resolve, Position, Justify): resolve posture, confidence, main reason, counter-argument, and what would change the read. Decide stays evidence-bound. Confidence is deterministic (`SportsEvaluator`); posture is a clamped enum (`play, pass, monitor, wait, compare, avoid`); the model owns the rationale text, not the number or the enum value.
- **Synthesize** (Integrate, Compose, Deliver): compress validated material into the final user-facing artifact. Deterministic (`SportsComposer`). Must introduce no new unsupported claims.

## 4. artifact contract sections

The internal decision artifact must account for the sections below. Each is mapped to the phase that owns it and the concrete field/record/component that backs it today, or a noted gap. The artifact's authoritative shape is the persisted `AgentRunExecutionResult` in `OutputJson` plus the retrieval diagnostics on `SportsRetrievalOutput`; the canonical cognitive narrative is `CognitiveProtocol`.

| # | section | owning phase | backed by today | status |
|---|---|---|---|---|
| 1 | game context | Perceive.Frame | `input` (competition, teams, date); `perceive.frame` | present |
| 2 | decision frame | Perceive.Aim | `perceive.aim`; `RunType=sports.matchup.analysis` | present |
| 3 | available signals | Perceive.Detect / Retrieve | `SportsRetrievalOutput.GroundedSignals[]` | present |
| 4 | missing signals | Perceive.Detect / Interrogate | `SportsRetrievalOutput.MissingSignals[]`; `SignalAvailability[].Status` | present |
| 5 | weak signals | Discern.Weigh | `SignalAvailability[].Quality == "usable"` (vs "strong") | present |
| 6 | proxy signals | Interrogate.Probe / Discern | `SignalFollowUpRecord.FallbackType == "lateral_proxy"` | partial: detected; explicit artifact label is a gap |
| 7 | signal quality assessment | Discern.Weigh | `SignalQualityEvaluator` -> `Quality`, `DecisionUse`, `ConfidenceEffect` | present |
| 8 | market read | Discern.Contrast | `discern.contrast`; market context blocks | present |
| 9 | rest/schedule read | Perceive.Frame / Discern | `BasketballScheduleContext`; `rest_schedule` availability | present |
| 10 | injury/availability | Perceive.Detect | none | planned placeholder (see section 5) |
| 11 | matchup context | Perceive.Frame / Discern.Contrast | `perceive.frame`, `discern.contrast` | present (narrative only) |
| 12 | conflict / counter-signal | Interrogate.Question / Discern.Stress | `interrogate.question` (counter-case), `discern.stress` | present |
| 13 | lean / posture | Decide.Resolve / Decide.Position | `Lean`, `LeanSide`, `decide.position` (clamped enum) | present |
| 14 | confidence rationale | Decide.Justify | `Confidence`, `ConfidenceBand`, `EvidenceRichness` (`SportsEvaluator`); `decide.justify` | present |
| 15 | watch-fors | Decide / Interrogate.Probe | `WatchFor`, `WhatWouldChangeTheRead`, `interrogate.probe` | present |
| 16 | final compact read | Synthesize.Deliver | `AgentRunResultDto` (Lean, Summary, Posture, CounterCase, WatchFor, ...) | present |

Backward-compatibility rule: this contract is additive over the existing artifact. New fields or labels are added only where they give immediate inspection or quality value, never by renaming the broad downstream contract (`SportsCognitivePhases`/`SportsAnalysisResponse.Phases`) and never by rewriting persisted records. Old persisted artifacts must remain readable through the existing `ProtocolView` projection (v1 legacy-only and v2/v3 canonical-native all project).

## 5. signal source strategy

Each signal should eventually carry the following seven facets. This is the durable per-signal spec the source-hardening slices write against. Today the platform records `Source`, `Quality`, `DecisionUse`, and `ConfidenceEffect` per signal; the remaining facets (backup, proxy, failure mode, pay-for-later) are doctrine to be encoded as sources are hardened.

For each signal: (1) primary source, (2) backup source, (3) cheap or free proxy, (4) failure mode, (5) confidence impact if missing, (6) decision impact if weak, (7) whether it is worth paying for later.

Current signals:

- **market** (spread). primary: the-odds-api. backup: a second book / scraped line (planned). proxy: none used today. failure mode: offseason, no event match, team-name mismatch -> null. confidence if missing: dampen (market is the primary directional signal). decision if weak (no sharp_public confirmation): graded "usable", directional-only, blocks aggressive posture. worth paying for: yes, market is the spine of the read.
- **sharp_public** (bet% vs money% split). primary: actionnetwork (unofficial). backup: none today. proxy: line movement (permanently `not_implemented` today). failure mode: early week / offseason / api failure -> null. confidence if missing: `block_aggressive_posture`. decision if weak: the lean becomes market-only. worth paying for: likely, it is the confirmation signal that unlocks a strong read.
- **rest_schedule** (days rest, back-to-back). primary: espn scoreboard. backup: league schedule api (planned). proxy: none. failure mode: cannot ground both teams -> null. confidence if missing: neutral. decision if weak: supporting context only, `support_cautiously`. worth paying for: low; public data is adequate.
- **starting_pitching** (MLB probable starters). primary: statsapi.mlb.com (free). backup: none needed. proxy: none. failure mode: starters not yet announced -> null. confidence if missing: neutral (single-signal MLB runs). decision if weak: supporting context, `support_cautiously`. worth paying for: no; the free source is authoritative.
- **injury/availability** (planned, section 4 item 10): no source wired. this is the most valuable missing signal for basketball and football. when added it must follow the proxy rule below and a missing injury read should be surfaced explicitly, not silently omitted.

## 6. proxy signal rules

A proxy signal can inform the artifact, but it must obey two non-negotiable rules:
1. **it must label itself as a proxy.** A proxy standing in for a missing primary (`FallbackType == "lateral_proxy"`, where `TriggeredBy` names the missing primary) must be presented as a proxy in the artifact, never as the primary signal it substitutes for.
2. **it must reduce confidence accordingly.** A proxy never grades "strong" and never grants `support`; it grades at most "usable" with `support_cautiously`, and it must not unlock an aggressive posture on its own.

Today the platform detects lateral proxies (`SignalFollowUpEvaluator`) and grades non-confirmed signals as "usable"/`support_cautiously`. The remaining gap is an explicit artifact-level proxy label (section 4 item 6); that is a named next slice, not this one.

## 7. quality gates

Deterministic, platform-owned, applied after evaluate and before compose (`SportsQualityChecker`), surfaced as `ArtifactQualityWarnings` (visible on the artifact inspection endpoint and `/dev/artifacts`, not in the customer DTO). Calibration flags (`confidence_high_for_partial_evidence`, etc.) remain a separate offline surface computed by the calibration scripts.

Existing gates (rules 1-5): counter_case required for play/monitor; watch_for required for monitor/wait; what_would_change required under partial grounding; `signals_used` must be a subset of grounded signals (with alias normalization); posture must be present on a successful run.

**New gate added in Decision Artifact Contract v1 (rule 6, anti-hype):** an aggressive `play` posture must not be asserted when the signal layer blocks it. When any signal carries `ConfidenceEffect == "block_aggressive_posture"` (today: a missing `sharp_public` confirmation) and the model still proposed `play`, the checker emits a warning. It changes neither the confidence number nor the posture value; it makes the risk inspectable. This is the deterministic Discern-layer anti-hype guard in code.

Planned gates (next slices): explicit proxy-label gate (item 6); weak-evidence-confidence gate that surfaces `confidence_high_for_partial_evidence` at runtime (deferred because it intersects the confidence surface and the clean-pass contract; it currently lives offline by deliberate prior decision); injury/availability missing-read gate once that signal exists.

## 8. next implementation slices

1. **Decision Artifact Contract v1 (this slice).** Define the contract (this doc), add the deterministic anti-hype gate (rule 6), and pin the existing signal-awareness guarantees with characterization tests. No persisted-shape change, no confidence-math change, no contract rename.
2. **Proxy label surfacing.** Add an explicit proxy label to the artifact when a lateral proxy informs the read, enforced by a quality gate. Additive field on the inspection surface.
3. **Injury/availability signal.** Wire a first injury/availability source (or a labeled proxy) behind the Tool Gateway, with a missing-read gate so an absent injury read is surfaced, not omitted.
4. **Runtime weak-evidence confidence gate.** Bring `confidence_high_for_partial_evidence` onto the runtime artifact as a warning (not a number change), reconciling it with the clean-pass contract.
5. **Calibration loop tightening.** Feed reconciled outcomes back into the quality gates and (only with sufficient evidence) the confidence tiers.

Future-architecture note: all of the above are designed to remain compatible with later PostgreSQL/pgvector memory, Kubernetes hosting, telemetry, and ML calibration. None of those are implemented here, and none of these slices take a choice that would make them harder: signal diagnostics stay structured and deterministic, the artifact stays additive, and the Tool Gateway remains the single seam for new sources.

## references

- `02 Platform/architecture/cognitive-factory/protocol-node-specs.md` (per-node contracts)
- `02 Platform/architecture/cognitive-factory/protocol-station-blueprint-v1.md` (station cards)
- `02 Platform/architecture/current-agent-run-contract.md` (persisted artifact + signal diagnostics)
- code: `dai/platform/dotnet/DevCore.Api/AgentRuns/` -- `SportsRetrievalOutput.cs`, `SignalQualityEvaluator.cs`, `SignalFollowUpEvaluator.cs`, `SportsQualityChecker.cs`, `SportsEvaluator.cs`, `CognitiveProtocolBuilder.cs`, `SportsComposer.cs`
