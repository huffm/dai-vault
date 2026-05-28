# Cognitive Protocol Node Specs v1

**date:** 2026-05-18
**status:** doctrine. operational node contracts for the Cognitive Protocol Runtime. runtime code unchanged by this slice.

## what this document is

`cognitive-protocol-runtime.md` names the four macro protocols plus Synthesize and their 15 micro-actions. `protocol-vocabulary-map.md` maps canonical names to legacy code fields. This document is the layer below both: an **operational spec for each of the 15 nodes** — what it reads, what it writes, what tools and scripts it may use, when a model call is allowed, what can go wrong, and what it must never do.

It is the contract a future code slice implements against. It does not describe 15 separate model calls. Of the 12 cognitive micro-actions, **11 are model-emitted** — produced by a **single analyze model call** in FastAPI as the `SportsAnalyzerProtocolSeed` (the model-owned protocol seed) — and **one, Interrogate.Probe, is deterministic .NET**; the Synthesize trio is deterministic .NET and is not among the 12. The platform completes the seed into the canonical `CognitiveProtocol` (adding Probe and Synthesize) via `CognitiveProtocolBuilder.FromAnalyzerProtocolSeed`. The node specs describe each node's *responsibility and boundary* regardless of how many calls implement them.

Read each node like a specialized cell: it has receptors (the artifact fields it reads), it secretes one product (the field it writes), and it fires only on specific stimuli (its signal dependencies). A node activates a fixed schema when its inputs arrive — it does not roam, choose its own inputs, or widen its own permissions. The biology is a memory aid for that boundedness; the architecture underneath is plain typed code.

## the runtime in one paragraph

A user request creates one shared **decision artifact**. Deterministic platform code (`AgentRunService`, `SportsRunArtifact`) moves that artifact through Perceive → Interrogate → Discern → Decide → Synthesize. **Retrieve is not a node** — it is platform plumbing (`SportsRetriever`, the typed HTTP clients, `SignalQualityEvaluator`, `SignalFollowUpEvaluator`) that runs before Perceive and stamps grounded evidence onto the artifact. Perceive *consumes* retrieval output; it does not perform retrieval.

## how to read a node spec

Every node below defines the same 11 facets:

1. **Purpose** — the single responsibility, in one or two sentences.
2. **Artifact fields read** — the scoped inputs. A node may read nothing else.
3. **Artifact fields written** — the scoped output. A node writes nothing else.
4. **Allowed tools** — the closed list of platform clients / evaluators / model calls the node may use.
5. **Allowed scripts / reflexes** — deterministic code that runs for this node without a model.
6. **Model-call rule** — whether the node's content is model-emitted (inside the shared analyze call) or fully deterministic, and the bound on it.
7. **Signal dependencies** — which retrieved signals the node needs.
8. **Quality checks** — the deterministic checks and calibration flags that police the node.
9. **Fallback behavior** — what the node does when its inputs are missing or degraded.
10. **Sports overlay** — per-competition notes for NBA, MLB, NFL, NCAAM, NCAAW, where the node differs by competition.
11. **Forbidden behavior** — what the node must never do.

## global rules

These hold for every node and override any node-level convenience:

- **The model never chooses its own permissions.** Allowed tools, fields read, and fields written are fixed by this spec. A model call fills text into pre-scoped fields; it does not select tools, widen data access, or write outside its node.
- **Retrieve is platform plumbing, not a node.** No cognitive node performs source fetching. Probe may *recommend* investigation; it does not retrieve.
- **Deterministic where determinism is honest.** Probe and the Synthesize trio are deterministic today and stay that way. Confidence and confidence band are deterministic (`SportsEvaluator`). The model owns judgment text, not numbers, not posture enums, not artifact shape.
- **Synthesize transforms; it never invents.** It plates what survived the prior protocols and adds no new facts.
- **Legacy vs canonical.** Node names here are canonical. The legacy code field each maps to is cited per node — see `protocol-vocabulary-map.md`. No code is renamed by this slice.
- **Calibration flags are not ArtifactQualityWarnings.** The "Quality checks" facet below lists two different surfaces. Deterministic runtime warnings (signals_used integrity, narrative drift) are produced by `SportsQualityChecker` and stored in `ArtifactQualityWarnings` (the `quality_warnings` export column). Calibration flags such as `confidence_high_for_partial_evidence`, `posture_aligned_with_partial_evidence`, and `frame_missing_rest_context` are computed offline by `dai/scripts/dev/sports/run-artifact-calibration.ps1` and are not stored on the artifact. A reviewer scanning `quality_warnings` alone will not see a calibration flag; the outcome reconciliation export carries the decision-relevant ones in its `derived_calibration_flags` column (added 2026-05-20).

## shared decision artifact — field reference

Platform-owned (stamped by retrieval / evaluation / quality / compose — not by a cognitive node):

| field | owner | meaning |
|---|---|---|
| `input` (competition, homeTeam, awayTeam, gameDate) | request | the matchup |
| `GroundedSignals[]`, `MissingSignals[]` | `SportsRetriever` | which signal categories were grounded vs missing |
| `SignalAvailability[]` | `SignalQualityEvaluator` | per-signal Status, Quality, DecisionUse, ConfidenceEffect |
| `SignalFollowUps[]` | `SignalFollowUpEvaluator` | per (parent, follow-up) signal diagnostics |
| `AnalyzerConfidence` | analyze call | the model's provisional confidence |
| `Confidence`, `ConfidenceBand`, `EvidenceRichness` | `SportsEvaluator` | calibrated confidence; band; grounded-signal count |
| `ArtifactQualityWarnings[]`, `PipelineSteps[]` | `SportsQualityChecker` / pipeline | deterministic warnings; stage metadata |
| `ArtifactVersion` | `SportsComposer` | `sports_decision_artifact_v2` |

Cognitive-node-owned (the 15 nodes write these; canonical names):

| protocol | node fields | legacy source field |
|---|---|---|
| Perceive | `perceive.detect`, `perceive.frame`, `perceive.aim` | `perceive.detect / frame / aim` |
| Interrogate | `interrogate.question`, `interrogate.probe`, `interrogate.verify` | `interrogate.balance` / *(deterministic)* / `interrogate.reframe` |
| Discern | `discern.weigh`, `discern.contrast`, `discern.stress` | `discern.filter` / `discern.listen` / `interrogate.stress`→`discern.test` |
| Decide | `decide.resolve`, `decide.position`, `decide.justify` | `decide.voice` / `decide.posture` / `decide.calibrate` |
| Synthesize | `synthesize.integrate`, `synthesize.compose`, `synthesize.deliver` | *(deterministic — `SportsComposer`)* |

## sports signal map

| competition | code | analyzer family | grounded signals retrieved today | notes |
|---|---|---|---|---|
| NBA | `nba` | basketball | `rest_schedule`, `market`; `sharp_public` attempted | `sharp_public` returns null in playoffs (provider limitation) |
| NCAAM | `ncaamb` | basketball | `rest_schedule`, `market` | no season-long injury/availability source |
| NFL | `nfl` | football | `market`; `sharp_public` attempted | `sharp_public` available in regular season |
| NCAAF | `ncaaf` | football | `market` | shares the football analyzer with NFL |
| MLB | `mlb` | mlb | `starting_pitching` | no `market` or `sharp_public` in the MLB pipeline today |
| NCAAW | — | — | — | **not a platform competition** — no `CompetitionCatalog` entry, no seeded teams, no retriever route. See Open Questions. |

`line_movement` is a named follow-up signal but is permanently `not_implemented` today.

---

# Perceive

Surface what is, name what is missing, aim attention at what matters. Perceive reads the retrieved evidence; it never retrieves.

## Detect

1. **Purpose** — Identify the available signals, anomalies, and missing context for this matchup.
2. **Artifact fields read** — `input`; `GroundedSignals[]`, `MissingSignals[]`, `SignalAvailability[]`; the competition context blocks (`FootballMarketContext`, `MlbStarterContext`, `BasketballScheduleContext`, `BasketballMarketContext`, `SharpPublicContext`).
3. **Artifact fields written** — `perceive.detect`.
4. **Allowed tools** — the shared analyze model call; read access to the already-stamped retrieval output. No retrieval clients (retrieval is complete before Perceive).
5. **Allowed scripts / reflexes** — none of its own; `SignalQualityEvaluator` and `SignalFollowUpEvaluator` have already run retrieval-side and stamped `SignalAvailability` / `SignalFollowUps`, which Detect reads as given.
6. **Model-call rule** — model-emitted inside the shared analyze call (legacy `perceive.detect[]`). Candidate to move to deterministic retriever-side extraction in a future slice.
7. **Signal dependencies** — all signal categories for the competition; Detect must name which grounded and which are missing.
8. **Quality checks** — missing signals must be named explicitly, not skipped; `SportsQualityChecker` normalizes `signals_used` aliases and checks grounded-set membership.
9. **Fallback behavior** — zero grounded signals → Detect states "no grounded signals; prior-only reasoning" and names nothing it does not have.
10. **Sports overlay** — NBA / NCAAM: detect `rest_schedule` + `market`, note `sharp_public` availability. NFL: detect `market`, note `sharp_public`. MLB: detect `starting_pitching` only. NCAAW: n/a (unsupported competition).
11. **Forbidden behavior** — invent a signal, overstate signal strength, claim form/injury/travel not in the context blocks, decide posture.

## Frame

1. **Purpose** — State the factual matchup context in plain language.
2. **Artifact fields read** — `input`; the retrieval context blocks; `perceive.detect`.
3. **Artifact fields written** — `perceive.frame`.
4. **Allowed tools** — the shared analyze model call.
5. **Allowed scripts / reflexes** — none.
6. **Model-call rule** — model-emitted (legacy `perceive.frame`). Candidate for retriever-side templating later.
7. **Signal dependencies** — `market` (spread), `rest_schedule`, `starting_pitching` as available.
8. **Quality checks** — Frame should name the spread when `market` is grounded and rest when `rest_schedule` is grounded; calibration flags `frame_missing_spread_context` and `frame_missing_rest_context` police this.
9. **Fallback behavior** — no grounded context → Frame states the matchup neutrally without inventing detail.
10. **Sports overlay** — NBA / NCAAM: rest days + spread. NFL: spread. MLB: probable starters. NCAAW: n/a.
11. **Forbidden behavior** — invent context, editorialize, state a lean or posture.

## Aim

1. **Purpose** — Name the factors that matter most for this decision.
2. **Artifact fields read** — `perceive.detect`, `perceive.frame`; `GroundedSignals[]`.
3. **Artifact fields written** — `perceive.aim`.
4. **Allowed tools** — the shared analyze model call.
5. **Allowed scripts / reflexes** — none.
6. **Model-call rule** — model-emitted (legacy `perceive.aim[]`).
7. **Signal dependencies** — the grounded signal set; Aim must point only at grounded factors.
8. **Quality checks** — aimed factors must be grounded; speculative factors are a guardrail violation.
9. **Fallback behavior** — thin evidence → Aim narrows to what is grounded (e.g. "market direction only").
10. **Sports overlay** — NBA / NCAAM: rest edge + market. NFL: market / spread. MLB: starter matchup. NCAAW: n/a.
11. **Forbidden behavior** — aim at ungrounded factors (injuries, form, travel) without grounding.

---

# Interrogate

Apply pressure to the initial read: name the open question, probe the gaps, verify against staged evidence.

## Question

1. **Purpose** — Name the strongest open question or counter-case against the emerging read.
2. **Artifact fields read** — `perceive.*`; `GroundedSignals[]`, `MissingSignals[]`, `SignalAvailability[]`.
3. **Artifact fields written** — `interrogate.question`.
4. **Allowed tools** — the shared analyze model call.
5. **Allowed scripts / reflexes** — none.
6. **Model-call rule** — model-emitted (legacy `interrogate.balance`).
7. **Signal dependencies** — the grounded set; the question must cite a specific signal or explicitly name the missing-signal limitation.
8. **Quality checks** — Interrogate guardrail v1.5: must cite a signal or the missing-signal limit; prohibited phrases ("could outperform", "has potential", "may surprise", "talent", "strong team", "recent performance"); calibration flag `counter_case_generic`.
9. **Fallback behavior** — thin evidence → name the limitation as the counter-case; do not invent a team trait.
10. **Sports overlay** — NBA / NCAAM: counter-case usually rest or market reliability. NFL: market-only reliability. MLB: starter-dependence of the lean. NCAAW: n/a.
11. **Forbidden behavior** — invent a counter-case, add new facts, decide the final stance.

## Probe

1. **Purpose** — Summarize the signal gaps worth investigating: which missing primary signals undercut the read.
2. **Artifact fields read** — `SignalFollowUps[]` (`SignalFollowUpRecord[]`).
3. **Artifact fields written** — `interrogate.probe`.
4. **Allowed tools** — none. No model call.
5. **Allowed scripts / reflexes** — `CognitiveProtocolBuilder.BuildProbe`. It identifies missing primaries (`Reason == "primary_signal_missing"`, or `DecisionUse == "missing_confirmation"`, or `FallbackType == "lateral_proxy"` → the `TriggeredBy` parent), dedups and orders them, and maps each to a doctrinal probe template.
6. **Model-call rule** — **deterministic. No model call.** Probe is built at compose time from signal follow-up records. (This is the rule, not an implementation detail.)
7. **Signal dependencies** — `SignalFollowUpRecord[]` from `SignalFollowUpEvaluator`. Doctrinal templates exist for `sharp_public`, `market`, `rest_schedule`, `starting_pitching`. `line_movement` is deliberately excluded (permanently `not_implemented`).
8. **Quality checks** — a signal with no doctrinal template is silently dropped, so Probe cannot fabricate. Output is deterministic and inspectable.
9. **Fallback behavior** — no follow-up records, no missing primary, or no templated signal → Probe is null and renders "Not recorded".
10. **Sports overlay** — NBA / NCAAM / NFL: the `sharp_public` template fires often (sharp/public frequently missing). MLB: `starting_pitching` is usually grounded, so Probe is usually null; the template fires only if a starter is unavailable. NCAAW: n/a.
11. **Forbidden behavior** — make a model call, fabricate injury/form/travel claims, emit a template for an uncharacterized signal, probe `line_movement`, recommend unrestricted retrieval.

## Verify

1. **Purpose** — Confirm or reject the emerging read by offering an alternate explanation tested against staged evidence.
2. **Artifact fields read** — `perceive.*`, `interrogate.question`; `SignalAvailability[]`.
3. **Artifact fields written** — `interrogate.verify`.
4. **Allowed tools** — the shared analyze model call.
5. **Allowed scripts / reflexes** — none.
6. **Model-call rule** — model-emitted (legacy `interrogate.reframe`).
7. **Signal dependencies** — the grounded set.
8. **Quality checks** — reframe guardrail v1.5: must not invoke hidden strengths, resilience, trends, form, travel, weather, injuries, or player availability without grounded context.
9. **Fallback behavior** — no grounded alternate exists → state that plainly rather than inventing one.
10. **Sports overlay** — NBA / NCAAM: rest or matchup-pace alternates. NFL: spread-origin alternates. MLB: starter-regression alternates. NCAAW: n/a.
11. **Forbidden behavior** — introduce ungrounded narrative, treat the reframe as a newly established fact.

---

# Discern

Judge evidence quality after collection: weigh signals, contrast them, stress-test the read.

## Weigh

1. **Purpose** — Grade signals by quality and decision-use; separate grounded evidence from weak or absent evidence.
2. **Artifact fields read** — `SignalAvailability[]`, `GroundedSignals[]`, `MissingSignals[]`.
3. **Artifact fields written** — `discern.weigh` (the narrative side). The deterministic grades already live on `SignalAvailability[]`.
4. **Allowed tools** — `SignalQualityEvaluator` (deterministic, retrieval-side); the shared analyze model call for the narrative only.
5. **Allowed scripts / reflexes** — `SignalQualityEvaluator` computes `Quality` (strong / usable / unavailable), `DecisionUse`, and `ConfidenceEffect` — including `block_aggressive_posture` when `sharp_public` is missing. This deterministic grading is authoritative.
6. **Model-call rule** — the deterministic backbone is authoritative; the model emits only a narrative `filter` sentence (legacy `discern.filter`) that must agree with the deterministic grades.
7. **Signal dependencies** — all signals; `market` quality is `strong` only when `sharp_public` co-grounds, otherwise `usable` / directional-only.
8. **Quality checks** — the narrative must name what is admitted, what is missing, and what is discounted (Discern guardrail v1.5).
9. **Fallback behavior** — no signals → Weigh states nothing is grounded; the deterministic grades reflect that.
10. **Sports overlay** — NBA / NCAAM: `market` usable not strong without `sharp_public`. NFL: same. MLB: only `starting_pitching` to weigh — single-signal runs. NCAAW: n/a.
11. **Forbidden behavior** — admit weak evidence as strong, treat interpretation as fact, let the model narrative override the deterministic quality grades.

## Contrast

1. **Purpose** — Interpret alignment or divergence between signals (market vs sharp/public vs situational).
2. **Artifact fields read** — `SignalAvailability[]`, `GroundedSignals[]`; `perceive.*`.
3. **Artifact fields written** — `discern.contrast`.
4. **Allowed tools** — the shared analyze model call.
5. **Allowed scripts / reflexes** — none.
6. **Model-call rule** — model-emitted (legacy `discern.listen`).
7. **Signal dependencies** — needs two or more grounded signals to contrast meaningfully; `market` vs `sharp_public` is the canonical contrast.
8. **Quality checks** — Contrast must not be null when external signal blocks were supplied; calibration flag `listen_missing_with_external_signal`.
9. **Fallback behavior** — a single grounded signal → state there is nothing to contrast it against.
10. **Sports overlay** — NBA / NCAAM: market vs rest, market vs sharp/public when present. NFL: market vs sharp/public. MLB: single-signal — contrast is usually limited or absent. NCAAW: n/a.
11. **Forbidden behavior** — invent divergence, treat one signal as corroboration of itself.

## Stress

1. **Purpose** — Name the fragility, key risk, or the condition under which the read fails. Canonical home of Stress is Discern (legacy code still emits it under `interrogate`).
2. **Artifact fields read** — `perceive.*`, `discern.weigh`, `discern.contrast`; `MissingSignals[]`.
3. **Artifact fields written** — `discern.stress`.
4. **Allowed tools** — the shared analyze model call.
5. **Allowed scripts / reflexes** — `CognitiveProtocolBuilder` selects a single source for canonical Stress: legacy `interrogate.stress`, falling back to legacy `discern.test`. No concatenation.
6. **Model-call rule** — model-emitted; the builder picks one source deterministically.
7. **Signal dependencies** — the grounded set; Stress frequently names a *missing* signal as the fragility.
8. **Quality checks** — Stress guardrail v1.5: must name a concrete fragility (missing `sharp_public`, large spread cover risk, market-only lean, equal rest limiting the schedule edge, missing injury/lineup data).
9. **Fallback behavior** — a fragility is always nameable; at minimum, "thin evidence base".
10. **Sports overlay** — NBA / NCAAM: market-only lean, equal rest. NFL: spread cover risk, market-only lean. MLB: starter-dependence, bullpen uncertainty. NCAAW: n/a.
11. **Forbidden behavior** — emit a vague fragility, fabricate an injury or form risk that is not grounded.

---

# Decide

Turn discerned evidence into a calibrated stance: resolve the read, set the posture, justify the calibration.

## Resolve

1. **Purpose** — Reconcile the weighed evidence into a single direction or non-direction, phrased without hype.
2. **Artifact fields read** — `discern.*`, `perceive.*`.
3. **Artifact fields written** — `decide.resolve`; contributes to the extracted `Lean` and `LeanSide`.
4. **Allowed tools** — the shared analyze model call.
5. **Allowed scripts / reflexes** — FastAPI validates and clamps `LeanSide` to `home`, `away`, or `null`; invalid model output is logged and clamped.
6. **Model-call rule** — model-emitted (legacy `decide.voice`); `Lean` / `LeanSide` are extracted and validated platform-side.
7. **Signal dependencies** — the grounded set.
8. **Quality checks** — no hype or certainty language; `LeanSide` clamp enforces a valid direction or null.
9. **Fallback behavior** — signals split → resolve to a non-direction ("signals are split"); `LeanSide` is null.
10. **Sports overlay** — direction is competition-specific but the resolve contract is identical across NBA / NCAAM / NFL / MLB. NCAAW: n/a.
11. **Forbidden behavior** — emit a pick or lock, use certainty language, claim a direction the evidence does not support.

## Position

1. **Purpose** — Set the decision posture from the niche posture vocabulary.
2. **Artifact fields read** — `discern.*`, `decide.resolve`; `Confidence`, `SignalAvailability[]` (especially `ConfidenceEffect = block_aggressive_posture`).
3. **Artifact fields written** — `decide.position` (the posture string).
4. **Allowed tools** — the shared analyze model call proposes; the platform validates.
5. **Allowed scripts / reflexes** — posture is validated against the closed enum `{ play, pass, monitor, wait, compare, avoid }` and clamped to null on any invalid value; `CognitiveProtocolBuilder` preserves the validated string exactly.
6. **Model-call rule** — the model proposes a posture, but Position is a **clamped enum**, not free text. The platform, not the model, is the final authority on the value.
7. **Signal dependencies** — posture must respect `block_aggressive_posture` — an aggressive posture is not warranted when `sharp_public` is missing.
8. **Quality checks** — calibration flags `posture_aligned_with_partial_evidence` and `aggressive_posture_with_partial_evidence`; a `posture absent` warning.
9. **Fallback behavior** — invalid or absent posture → null.
10. **Sports overlay** — the posture vocabulary is the same across NBA / NCAAM / NFL / MLB today; niche-specific posture vocabularies are doctrine but not implemented. NCAAW: n/a.
11. **Forbidden behavior** — treat posture as a pick, label it "Pick" in any UI, use lock language, choose an aggressive posture when the signal layer blocks it, invent a posture value outside the enum.

## Justify

1. **Purpose** — One-sentence rationale for why the stated confidence fits the evidence.
2. **Artifact fields read** — `Confidence`, `ConfidenceBand`, `EvidenceRichness`, `AnalyzerConfidence`; `discern.weigh`.
3. **Artifact fields written** — `decide.justify`.
4. **Allowed tools** — the shared analyze model call for the sentence; `SportsEvaluator` for the number.
5. **Allowed scripts / reflexes** — `SportsEvaluator` computes the calibrated `Confidence` (dampening by grounded-signal count, clamped per grounding tier) and the `ConfidenceBand` (`>= 0.70` high, `>= 0.50` medium, else low). The model does not own the number.
6. **Model-call rule** — the model emits the **rationale sentence only**; the confidence value and band are deterministic and authoritative.
7. **Signal dependencies** — the grounded-signal count drives the calibration tier.
8. **Quality checks** — calibration flag `confidence_high_for_partial_evidence` (`EvidenceRichness < 3` and `Confidence >= 0.70`).
9. **Fallback behavior** — zero grounded signals → confidence is dampened and clamped low; Justify states the prior-only basis.
10. **Sports overlay** — NBA / NCAAM: partial-grounding tier when `sharp_public` is missing. NFL: similar. MLB: single-signal calibration. NCAAW: n/a.
11. **Forbidden behavior** — let the model override the calibrated confidence, let stated confidence exceed the evidence, restate the number instead of justifying it.

---

# Synthesize

The final layer. Synthesize is platform-operational, not simulated cognition. It is **fully deterministic** — `SportsComposer` and `CognitiveProtocolBuilder`, no model call — and is not counted among the 12 cognitive micro-actions.

## Integrate

1. **Purpose** — Combine validated material from the prior protocols without adding new claims.
2. **Artifact fields read** — every prior node output on `SportsRunArtifact` (analyzer output, retrieval output, evaluator result).
3. **Artifact fields written** — `synthesize.integrate` (a fixed informational description); assembles the `AgentRunExecutionResult` fields.
4. **Allowed tools** — none. Deterministic `SportsComposer.Compose`.
5. **Allowed scripts / reflexes** — `SportsComposer.Compose`; `CognitiveProtocolBuilder.FromLegacy(phases, signalFollowUps)`.
6. **Model-call rule** — **no model call.**
7. **Signal dependencies** — none; Integrate operates on already-resolved artifact material.
8. **Quality checks** — must introduce no new claim; `ArtifactQualityWarnings` are carried through unchanged.
9. **Fallback behavior** — an analyze-stage failure routes to `ComposeFailedRun`: a minimal artifact, `CognitiveProtocol` null, `ArtifactVersion` still stamped.
10. **Sports overlay** — competition-agnostic.
11. **Forbidden behavior** — invent facts, add new claims, override evidence quality.

## Compose

1. **Purpose** — Assemble the final decision artifact shape.
2. **Artifact fields read** — the integrated material.
3. **Artifact fields written** — `synthesize.compose`; the `AgentRunExecutionResult` / `OutputJson` shape, `ArtifactVersion`, the canonical `CognitiveProtocol` block (dual-emitted alongside legacy `CognitivePhases`).
4. **Allowed tools** — none. `SportsComposer` + `CognitiveProtocolBuilder`.
5. **Allowed scripts / reflexes** — stamps `ArtifactVersion = sports_decision_artifact_v2`; builds the canonical `CognitiveProtocol` from the legacy phases; runs `BuildProbe` (the Probe node).
6. **Model-call rule** — **no model call.**
7. **Signal dependencies** — none.
8. **Quality checks** — deterministic shape; canonical and legacy stay aligned because both derive from the same single analyzer call.
9. **Fallback behavior** — failure path still stamps `ArtifactVersion`; `CognitiveProtocol` stays null.
10. **Sports overlay** — competition-agnostic.
11. **Forbidden behavior** — invent fields, make a model call, let the artifact schema drift from the contract.

## Deliver

1. **Purpose** — Return the consumable result and persist the full artifact.
2. **Artifact fields read** — the composed artifact.
3. **Artifact fields written** — `synthesize.deliver`; the user-facing `AgentRunResultDto` (compact delivery fields); the persisted `OutputJson`; `EvidenceRichness` from the grounded-signal count.
4. **Allowed tools** — none. `SportsComposer` for mapping; `AgentRunsController` persists `OutputJson` and the `AgentRun` row.
5. **Allowed scripts / reflexes** — maps compact delivery fields (`Lean`, `Summary`, `Posture`, `CounterCase`, `WatchFor`, `WhatWouldChangeTheRead`) to the DTO; `EvidenceRichness` is the retriever's grounded count, not model output.
6. **Model-call rule** — **no model call.**
7. **Signal dependencies** — none.
8. **Quality checks** — the user-facing DTO excludes internal quality fields (`MissingSignals`, `ArtifactQualityWarnings`, `SignalAvailability`, `SignalFollowUps`); those stay in `OutputJson` and the artifact inspection endpoint.
9. **Fallback behavior** — failure → `status = failed`, `FailureArtifact` persisted into `OutputJson`, pipeline metadata preserved.
10. **Sports overlay** — competition-agnostic; the UI labels `Posture` as **Read Stance**, never "Pick".
11. **Forbidden behavior** — leak raw prompts / provider metadata / API keys, invent delivery content, relabel posture as a pick.

---

## sports overlay — consolidated

- **NBA** (`nba`) and **NCAAM** (`ncaamb`) share the basketball analyzer family. Both ground `rest_schedule` + `market`; both attempt `sharp_public`, which is frequently missing (in NBA, null in playoffs by provider limitation). NCAAM additionally has no season-long injury/availability source — do not force injury grounding for it.
- **NFL** (`nfl`) uses the football analyzer, grounds `market`, attempts `sharp_public` (available in the regular season). **NCAAF** (`ncaaf`) shares the football analyzer and grounds `market`; the node contracts above apply to NCAAF identically to NFL.
- **MLB** (`mlb`) grounds only `starting_pitching` today — single-signal runs, `EvidenceRichness = 1`, `Contrast` usually limited, `Probe` usually null. No `market` or `sharp_public` in the MLB pipeline.
- **NCAAW** — women's college basketball is **not a platform competition**. There is no `CompetitionCatalog` entry, no seeded teams, and no retriever route. No node overlay can be written for it without first adding the competition. See Open Questions.

## open questions

1. **NCAAW is out of scope.** The slice brief asks for NCAAW overlay notes, but NCAAW is not a supported competition. Decide whether to (a) add NCAAW to `CompetitionCatalog` + seed teams + add a retriever route as a separate slice, or (b) drop NCAAW from the cognitive node scope until that exists. This doc assumes (b) and flags the gap.
2. **Detect / Frame ownership.** Both are model-emitted today but the runtime doctrine allows moving them to deterministic retriever-side extraction. The node specs assume "model today"; a future slice should decide when Detect/Frame move and update facet 6.
3. **One call vs many.** The 12 cognitive micro-actions are produced by one analyze call today. Whether nodes are ever split into separate calls or skill packs (per `cognitive-skill-pack-architecture-v1.md`) is undecided; the per-node contracts here hold either way.
4. **Discern.Weigh dual ownership.** Weigh has a deterministic backbone (`SignalQualityEvaluator`) and a model narrative (legacy `filter`). This doc treats the deterministic grades as authoritative and the narrative as subordinate; a future slice should confirm that ordering in code (`SportsQualityChecker`).
5. **Probe template coverage.** Probe templates exist for only four signals; adding any new grounded signal requires a new doctrinal template, or Probe silently drops it.

## references

- `02 Platform/decisions/0004-cognitive-protocol-runtime.md`
- `02 Platform/architecture/cognitive-factory/cognitive-protocol-runtime.md`
- `02 Platform/architecture/cognitive-factory/protocol-vocabulary-map.md`
- `02 Platform/architecture/cognitive-factory/phases/` — per-phase responsibility docs
- `02 Platform/architecture/sports-cognitive-worker-model.md`
- `02 Platform/architecture/current-agent-run-contract.md`
- `02 Platform/architecture/current-sports-analysis-flow.md`
