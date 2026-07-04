---
title: "Synthesize Station Application Mapping v1"
type: "execution-pattern"
date: "2026-07-04"
status: "draft"
project: "DAI"
slice: "Synthesize Station Application Mapping v1"
repos:
  dai: "unchanged"
  dai-vault: "docs-only"
tags:
  - protocol
  - synthesize
  - cognitive-factory
  - buyer-safety
  - architecture
related:
  - "06 Execution/patterns/decide-station-application-mapping-v1.md"
  - "02 Platform/architecture/cognitive-factory/protocol-node-specs.md"
  - "02 Platform/architecture/cognitive-factory/phases/synthesize.md"
  - "04 Products/sports-v1/buyer-copy-safety-v1.md"
  - "04 Products/sports-v1/buyer-packaging-source-of-truth-v1.md"
  - "02 Platform/architecture/current-agent-run-contract.md"
  - "06 Execution/patterns/okf-documentation-review-guide-v1.md"
---

# Synthesize Station Application Mapping v1

## purpose

Map DAI's current sports decision pipeline into the Cognitive Protocol's **Synthesize** station and its three
operations -- Integrate, Compose, Deliver. Define what Synthesize consumes from `DecisionPosture`, what it produces
for internal and buyer-safe surfaces, what residue it preserves, and how it prevents overstatement. Design-only: no
runtime, prompt, selection, confidence, advertised-strength, evidence-sufficiency, buyer-copy, buyer-artifact,
metrics, or schema change. This slice completes the five-station protocol-mapping thread.

## context

Synthesize is the final layer: Perceive -> Interrogate -> Discern -> Decide -> **Synthesize**. The prior slice
matured Decide into a `DecisionPosture` output. Inherited principle: **Synthesize does not decide, does not
strengthen posture; it integrates, constrains, formats, and delivers.** Two honest truths shape this mapping:

1. **Synthesize is fully deterministic today** (`SportsComposer`, no model call). It "plates what is already true --
   it does not cook new evidence" (`phases/synthesize.md`). Integrate/Compose/Deliver are platform operations, not
   cognitive micro-actions, and are not among the 12.
2. **Buyer-safe copy is not centralized in the .NET Synthesize layer today.** It is realized at three deterministic
   boundaries: the **FastAPI top-level buyer-copy sanitizer** (on `lean`/`summary`/`factors`/`counter_case`/
   `watch_for`/`what_would_change_the_read`), the **Angular buyer-signal mapper** (aggregates internal gaps into a
   single `Confirmation strength` row), and the **`AgentRunResultDto` omission** of internal quality fields
   (`SignalAvailability`, `SignalFollowUps`, `MissingSignals`, `ArtifactQualityWarnings`). `SportsComposer` maps
   compact fields and enforces evidence/confidence ownership; it does not itself rewrite copy.

So the *canonical* protocol model assigns buyer-safe composition to Synthesize.Compose, but the *current
implementation* distributes it across the analyzer sanitizer + frontend mapper + DTO omission. This doc maps both;
it changes neither.

## scope

**Included:** a Synthesize mental model; the Synthesize doctrine + current-pipeline synthesis field map; Integrate/
Compose/Deliver mapping (inputs from DecisionPosture, current runtime analogs, deterministic rules, LLM role,
outputs, residue, forbidden); a proposed `SynthesizedArtifactPackage` contract; a deterministic synthesis rule
taxonomy with current-vs-future implementability; the LLM role boundary; the buyer-projection boundary.

**Excluded:** any runtime/code/prompt/selection/confidence/advertised-strength/evidence-sufficiency/buyer-copy/
buyer-artifact/metrics/schema change; implementing `SynthesizedArtifactPackage` or a Synthesize runner; changing
buyer projection behavior (mapped only); altering the node specs or `SportsComposer`.

## synthesize mental model

*Synthesize plates what is already true.* It takes the committed `DecisionPosture`, assembles the internal artifact,
derives the buyer-safe surface under strict language rules, and delivers each to the correct surface -- adding no
claim, hiding no uncertainty, and **never intensifying** language. It may soften or constrain; it may never
strengthen. Posture stays **Read Stance**, never "Pick". Final confidence stays the deterministic
`AggregateConfidence`; `EvidenceRichness` stays the retriever's grounded count -- Synthesize surfaces these, it does
not recompute them.

## synthesize doctrine inventory (Phase 1)

| path | purpose | status | relevance to I/C/D | touched |
|---|---|---|---|---|
| `phases/synthesize.md` | Synthesize phase doctrine; "Manifest" synonym; deterministic, no model call | v1 doctrine | high (all three) | no |
| `protocol-node-specs.md` (Synthesize) | 11-facet Integrate/Compose/Deliver contracts; deterministic | doctrine | high | no |
| `CognitiveProtocolBuilder.cs` | Synthesize trio = platform-operational constants; builds canonical `CognitiveProtocol`, `ArtifactVersion` | implemented | Compose representation | no |
| `SportsComposer.Compose` | assembles `AgentRunExecutionResult`; maps compact `AgentRunResultDto`; evidence/confidence ownership | implemented | Integrate/Compose/Deliver | no |
| FastAPI top-level buyer-copy sanitizer | deterministic sanitize of buyer fields; forbidden-phrase filter | implemented (`buyer-copy-safety-v1`) | Compose (buyer copy) | no |
| Angular buyer-signal mapper (`buyer-signal-summary.ts`) | aggregates internal gaps into `Confirmation strength`; hides non-grounded rows | implemented | Compose/Deliver (buyer render) | no |
| `AgentRunResultDto` | compact delivery DTO; omits internal quality fields | implemented (`current-agent-run-contract`) | Deliver (omission) | no |
| `buyer-copy-safety-v1.md` | buyer language allow/forbid rules; internal-vs-buyer split | doctrine | high (Compose rules) | no |
| `buyer-packaging-source-of-truth-v1.md` | buyer packaging source-of-truth boundary | doctrine | Deliver boundary | no |

## current pipeline synthesis field map (Phase 2)

`origin` = model-authored | det-projection (deterministic projection) | buyer-projection | internal-only.
`S` = Synthesize relevance; `action` = read / write / transform / omit / preserve (buyer-side).

| field | producer | origin | persist | buyer | S | action |
|---|---|---|---|---|---|---|
| internal artifact summary | model + `SportsComposer` | model + compose | yes | via buyer summary | assembles | write |
| buyer artifact summary | model → FastAPI sanitizer | buyer-projection | yes | **yes** | derives | transform |
| rationale (`decide.justify`) | model | model-authored | yes | internal | preserves | preserve |
| key factors | model → sanitizer | model + buyer-projection | yes | yes | derives | transform |
| lean wording | model → sanitizer | model + buyer-projection | yes | yes | derives | transform |
| confidence wording | derived from `ConfidenceBand` | det-projection | yes | yes | phrases | transform (never recompute) |
| confidence (number) | `SportsEvaluator` `AggregateConfidence` | det | yes | yes | preserves | preserve |
| advertisedStrength | buyer projection | buyer-projection | yes/projected | **yes** | preserves | preserve (never change) |
| evidence-sufficiency wording | sufficiency gate → copy | det-projection | yes | yes | phrases | transform |
| source-depth chips | Angular `buyer-signal-summary` | buyer-projection | render | **yes** | aggregates | transform (grounded rows + `Confirmation strength`) |
| route/provenance metadata | route decision | internal-only | yes | **no** | keeps internal | omit (buyer) |
| promptSource / selectedPromptPath / recipe* | route decision / registry | internal-only | yes | no | keeps internal | omit |
| safety disclaimers | Angular / copy | det-projection | render | yes | applies | write |
| limitations / caveats | Decide `limitingFactors` | model + det | yes | yes (softened) | propagates | transform |
| no-decision language | Angular no-lean fallback | det-projection | render | yes | phrases | transform (`mixed read` / `no clear lean`) |
| buyer-safe terminology | prompt + sanitizer | buyer-projection | yes | yes | enforces | transform |
| fields omitted from buyer | `AgentRunResultDto` | det | n/a | **no** | omits | omit (`SignalAvailability`/`SignalFollowUps`/`MissingSignals`/`ArtifactQualityWarnings`) |
| delivery route/surface | `AgentRunsController` / `/dev/artifacts` | det | n/a | mixed | routes | write |

## synthesize station mapping (Phase 4)

Expected posture: **Integrate deterministic-heavy; Compose may be LLM-assisted for language but is constrained by
buyer-safety rules; Deliver deterministic. Buyer-safe copy belongs here, not Decide. Synthesize may soften/constrain
but never intensify.**

### Integrate

- **purpose:** combine `DecisionPosture` with route/provenance, evidence context, caveats, limitations, and delivery
  constraints into one internal package -- adding no new claim.
- **inputs from DecisionPosture:** `decisionStatus`, `leanSide`, `postureStrength`, `confidence`/`confidenceBand`,
  `evidenceSufficiency`, `primarySupportSignals`, `limitingFactors`, `acceptedContrasts`, `activeStressors`,
  `buyerSafetyConstraints`, `unresolvedQuestions`, route/provenance metadata, `residueNotes`.
- **current runtime analogs:** `SportsComposer` reading `SportsRetrievalOutput` + `SportsAnalysisResponse` +
  `EvaluatorOutput` from the artifact; `CognitiveProtocolBuilder` completing the canonical protocol.
- **deterministic rules:** caveat propagation, unsupported-claim exclusion, internal-only provenance (see taxonomy).
  Evidence/confidence ownership preserved (`EvidenceRichness` from retriever, `Confidence` from evaluator).
- **optional LLM role:** none required today (Synthesize is deterministic); any future draft language is Compose's,
  not Integrate's.
- **outputs:** the integrated internal package (basis for the artifact + the buyer projection).
- **residue:** what posture/evidence/caveats/provenance were carried forward, and what was flagged internal-only.
- **forbidden:** add a new claim; strengthen posture; recompute confidence; drop a mandatory caveat or a
  `buyerSafetyConstraint`.

### Compose

- **purpose:** create the internal artifact text/fields and the buyer-safe copy from the integrated package, under
  strict language rules.
- **inputs from Integrate:** the integrated package.
- **current runtime analogs:** `SportsComposer.Compose` (internal `AgentRunExecutionResult` + compact fields); the
  **FastAPI buyer-copy sanitizer** and **Angular buyer-signal mapper** (buyer surface). Buyer composition is
  distributed today, not centralized in `SportsComposer`.
- **deterministic rules:** buyer-safe terminology, no-pick/no-tout language, posture-strength ceiling,
  confidence-expression, evidence-sufficiency expression, no-decision phrasing, source-depth chip, field omission
  (see taxonomy). These bound whatever language is produced.
- **optional LLM role:** draft internal rationale language and buyer-safe explanatory copy, summarize support/
  limits, phrase caveats, simplify technical language -- **all constrained by the buyer-safety rules and grounded in
  DecisionPosture**. (Today the analyzer model drafts buyer fields and the deterministic sanitizer bounds them; a
  future centralized Compose could do the same in one place.)
- **outputs:** `internalSummary`/`internalRationale`; `buyerSafeSummary`/`buyerSafeStance`/`buyerSafeEvidenceNotes`/
  `buyerSafeLimitations`.
- **residue:** the internal-vs-buyer text pair and which rules constrained the buyer copy.
- **forbidden:** produce "pick"/"lock"/"best bet"/tout language; expose internal gap terms (`sharp_public`,
  "missing signal", "fallback failed", "probe required") in buyer copy; intensify posture or confidence wording;
  convert a no-decision into a lean.

### Deliver

- **purpose:** send/expose the correct output surface (internal vs buyer) while preserving boundaries, and record
  what was delivered.
- **inputs from Compose:** the composed internal + buyer surfaces.
- **current runtime analogs:** `AgentRunsController` returns compact `AgentRunResultDto` to the frontend; the
  artifact inspection endpoint (`GET /api/agent-runs/{id}/artifact`) + `/dev/artifacts` expose the internal surface;
  `AgentRunService` persists `OutputJson`.
- **deterministic rules:** internal-only provenance, route-metadata omission for buyer, field omission,
  delivery-surface routing, needs-review routing (see taxonomy). Deterministic today.
- **optional LLM role:** none. Deliver is deterministic.
- **outputs:** `deliverySurface`, `deliveryStatus`; the persisted `OutputJson`; the returned DTO.
- **residue:** what surface received what, which fields were omitted, delivery confirmation.
- **forbidden:** leak internal route/provenance or raw prompts to the buyer; surface an internal quality field on
  the buyer DTO; relabel posture as a "Pick".

**Synthesize's boundary in one line:** it formats and constrains a decision already made; it never re-decides,
re-computes, or intensifies.

## proposed SynthesizedArtifactPackage contract (Phase 3)

A read-model capturing the internal + buyer surfaces Synthesize produces. **Docs-only; not implemented; not
runtime.** It preserves the decision, references (never recomputes) confidence, and keeps internal provenance
separate from buyer copy. `det` = deterministic, `llm` = LLM-authored (grounded, rule-bounded).

| field | type | producer | consumer | det/llm | persisted | buyer-visible |
|---|---|---|---|---|---|---|
| agentRunId / sourceProvider / externalGameId | ids | platform | audit/delivery | det | yes/future | no |
| decisionStatus | decision \| no_decision \| needs_review | Decide (preserved) | Deliver | det | future | no (drives phrasing) |
| leanSide | home \| away \| null | Decide (preserved) | Compose | det | yes | via buyer copy |
| postureStrength | none \| slight \| measured \| strong | Decide (preserved) | Compose | det | future | via buyer copy |
| confidence | number | `SportsEvaluator` | Compose | **det (referenced)** | yes | yes |
| confidenceBand | high \| medium \| low | `SportsEvaluator` | Compose | det | yes | phrased |
| evidenceSufficiency | enum/int | sufficiency gate | Compose | det | yes | phrased |
| advertisedStrength | enum | buyer projection | Compose | det | yes/projected | **yes (unchanged)** |
| internalSummary | string | Compose | internal artifact | det + llm | future | no |
| internalRationale | string | Compose (from justify) | internal artifact | llm (grounded) | yes | no |
| primarySupportSignals[] | string[] | Integrate (from Decide) | Compose | det | future | aggregated |
| limitingFactors[] | string[] | Integrate (from Decide) | Compose | det | future | softened |
| acceptedContrasts[] | ContrastFinding[] | Integrate | internal | det | future | no |
| activeStressors[] | StressFinding[] | Integrate | internal | det | future | no |
| buyerSafetyConstraints[] | string[] | Decide (preserved) | Compose (honors) | det | future | drives buyer |
| buyerSafeSummary | string | Compose | buyer artifact | llm (rule-bounded) | yes | **yes** |
| buyerSafeStance | string | Compose | buyer artifact | det + llm | yes | **yes (Read Stance)** |
| buyerSafeEvidenceNotes[] | string[] | Compose | buyer artifact | det + llm | render | **yes** |
| buyerSafeLimitations[] | string[] | Compose | buyer artifact | llm (rule-bounded) | render | **yes** |
| omittedInternalFields[] | string[] | Deliver | audit | det | future | no |
| routeProvenanceInternal | object | Integrate | internal only | det | yes | **no** |
| deliverySurface | internal \| buyer \| both | Deliver | routing | det | future | no |
| deliveryStatus | delivered \| failed \| withheld | Deliver | audit | det | future | no |
| residueNotes | string | all | audit/calibration | det | future | no |

**Guarantees:** does not redefine the decision; does not recompute confidence; does not strengthen posture;
preserves `buyerSafetyConstraints`; keeps `routeProvenanceInternal` out of buyer copy. Superset **view**; it does not
replace `AgentRunExecutionResult`, `AgentRunResultDto`, `OutputJson`, or the canonical `CognitiveProtocol`.

## deterministic synthesis rule taxonomy (Phase 5)

`blocks` = can withhold/route delivery; `constrains` = bounds language/fields. Implementability: **impl** = enforced
today; **deriv** = computable from current fields but not centralized; **future** = needs new capture/state.

| category | signal read | rule type | output | blocks | constrains | feeds | impl |
|---|---|---|---|---|---|---|---|
| buyer-safe terminology | buyer fields | allow-list phrasing | safe copy | no | yes | Compose | **impl** (FastAPI sanitizer + prompt) |
| no-pick / no-tout language | buyer fields | forbidden-phrase filter | scrubbed copy | no | yes | Compose | **impl** (sanitizer forbidden phrases; "Read Stance") |
| posture-strength ceiling | `postureStrength`/`aggressivePostureBlocked` | ceiling (never intensify) | capped stance | no | yes | Compose | deriv |
| confidence-expression | `confidenceBand` | phrase map (measured/cautious) | confidence wording | no | yes | Compose | deriv (number stays `AggregateConfidence`) |
| evidence-sufficiency expression | `evidenceSufficiency` | phrase map | sufficiency wording | no | yes | Compose | deriv |
| no-decision phrasing | `decisionStatus=no_decision` / null lean | phrase | "mixed read / no clear lean" | no | yes | Compose | **impl** (Angular no-lean fallback) |
| needs-review phrasing | `decisionStatus=needs_review` | route/phrase | internal-hold | routes | yes | Deliver | future (needs_review not a runtime state today) |
| internal-only provenance | route/provenance | omit-from-buyer | internal-only | no | yes | Integrate/Deliver | **impl** (protocol object not sanitized; DTO omits) |
| route-metadata omission (buyer) | `observed/selectedDataRegime`/`recipe*`/`promptSource` | omission | omitted | no | yes | Deliver | **impl** (not in `AgentRunResultDto`) |
| source-depth chip | `SignalAvailability` grounded | aggregate render | grounded rows + `Confirmation strength` | no | yes | Compose | **impl** (`buyer-signal-summary.ts`) |
| caveat propagation | `limitingFactors`/`buyerSafetyConstraints` | propagate (softened) | buyer limitations | no | yes | Integrate/Compose | deriv |
| unsupported-claim exclusion | `unsupportedClaims` | exclude | dropped claims | no | yes | Compose | deriv |
| field omission | internal quality fields | DTO omission | omitted | no | yes | Deliver | **impl** (DTO excludes `SignalAvailability`/`SignalFollowUps`/`MissingSignals`/`ArtifactQualityWarnings`) |
| delivery-surface routing | target surface | route | internal vs buyer | routes | no | Deliver | **impl** (DTO → frontend; artifact endpoint → `/dev`) |

## LLM role boundary (Phase 6)

**Allowed:** draft internal rationale language; draft buyer-safe explanatory copy; summarize support signals and
limiting factors; phrase caveats clearly; simplify technical language for buyer surfaces.

**Forbidden:** change `decisionStatus`, `leanSide`, `confidence`, or `postureStrength` (never upward); remove
deterministic caveats or `buyerSafetyConstraints`; invent evidence; convert a no-decision into a lean; produce
"pick"/"lock"/"best bet"/tout language; expose internal route/provenance when not allowed; alter calibration
conclusions; promote/enable registry routing.

**Grounding requirements:** every composed claim maps to DecisionPosture support/limits/stressors or verified
evidence; unsupported claims are excluded; buyer copy preserves uncertainty; buyer copy uses read-stance language;
internal and buyer surfaces remain distinct.

## buyer projection boundary (Phase 7)

- **Is buyer projection part of Synthesize or downstream?** Conceptually it is **Synthesize.Compose + Deliver**. In
  the current implementation it is realized **downstream and distributed** -- FastAPI sanitizer (buyer text) +
  Angular buyer-signal mapper (buyer render) + `AgentRunResultDto` omission -- rather than inside `SportsComposer`.
  A future slice could centralize it into Synthesize.Compose; **this slice does not change it.**
- **Deterministic buyer projections:** confidence/band wording, source-depth chips (`Confirmation strength`),
  no-lean phrasing, field omission -- all deterministic today.
- **Composed (language) buyer fields:** `summary`, `lean`, `factors`, `counter_case`, `watch_for`,
  `what_would_change_the_read` -- model-drafted, then deterministically sanitized.
- **Internal fields that must never cross to buyer:** `SignalAvailability`, `SignalFollowUps`, `MissingSignals`,
  `ArtifactQualityWarnings`, `interrogate.probe`, raw signal keys (`sharp_public`), fallback/equivalence fields,
  route/provenance (`observedDataRegime`, `selectedPromptPath`, `recipe*`, `promptSource`, `fallbackReason`).
- **Route/provenance representation:** kept in `OutputJson` / provenance / `/rows` for internal calibration; **never**
  surfaced on the buyer DTO.
- **Source depth shown safely:** grounded rows shown; missing/unavailable/proxy rows aggregated into one
  `Confirmation strength` row -- never "Not available" or a named missing signal.
- **Confidence / advertised strength phrasing:** confidence expressed as "measured"/"cautious" band language over
  the deterministic number; advertised strength unchanged. No betting-edge intensifiers.

## safety / non-actions

Docs-only. **Not changed:** runtime code; prompt text; prompt-selection behavior; registry routing (stays
default-off); confidence formula; advertised strength; evidence sufficiency; buyer copy; buyer artifact output;
metrics denominator; DB schema; delivery behavior. **Not done:** paid model calls (0); sports runs (0);
reconciliation writes (0); external sports API calls (0); DB migrations/backfills (0); no
`SynthesizedArtifactPackage` or Synthesize runner implemented; no buyer-projection change; no dormant runner wired
into production. Node specs, `phases/synthesize.md`, `SportsComposer`, and the buyer-safety layer are referenced and
unchanged. v8 generation budget remains closed at 10.

## next recommended slice

If the pending v8 cohort games are **Final** on StatsAPI: **Settle-and-Reassess Registry-Routed v8 Cohort** (the
standing runtime priority) -- settle each via the reconciliation residue contract. If they are **not** Final:
**Cognitive Protocol Mapping Closeout v1** -- reconcile the five station-mapping docs (Discern mapping + micro-
protocol, Decide, Synthesize) into one index, confirm the DiscernAssessment -> DecisionPosture ->
SynthesizedArtifactPackage contract chain is internally consistent, and record the minimum-viable read-only
implementation path across all stations.

## related

- [[decide-station-application-mapping-v1]] -- the `DecisionPosture` this station consumes.
- [[protocol-node-specs]] -- authoritative Synthesize node contracts (deterministic), unchanged here.
- `02 Platform/architecture/cognitive-factory/phases/synthesize.md` -- Synthesize doctrine ("Manifest").
- [[buyer-copy-safety-v1]] -- the buyer language allow/forbid rules Compose enforces.
- `02 Platform/architecture/current-agent-run-contract.md` -- the `AgentRunResultDto` omission boundary.
