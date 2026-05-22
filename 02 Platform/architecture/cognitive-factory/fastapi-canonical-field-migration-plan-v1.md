# FastAPI Canonical Field Migration Plan v1

**date:** 2026-05-22
**status:** plan. no runtime code changed by this slice. defines the lockstep migration that renames the analyzer's legacy cognitive-phase fields to canonical protocol fields.
**scope:** the careful, sequenced plan to move the FastAPI analyzer (and the .NET wire contract that mirrors it) from legacy `CognitivePhases` field names to the canonical Cognitive Protocol vocabulary, without breaking persistence, the read-side projection, calibration, or the product output.

## what this document is

ProtocolRegistry v0 and the Protocol Registry Startup Guard v1 are shipped: the runtime has 15 canonical station cards and validates station/tool alignment at boot. But the analyzer still emits legacy field names (`interrogate.balance`, `discern.filter`, `decide.voice`, ...), and `.NET` maps them into the canonical `CognitiveProtocol` at compose time via `CognitiveProtocolBuilder`. This plan defines how to make FastAPI emit canonical names directly, in lockstep across the prompt, the Pydantic models, and the `.NET` wire contract, so the legacy-to-canonical name remap stops being a per-run translation step.

It is a planning slice. It changes no FastAPI prompt, no Pydantic model, no `.NET` contract, no `CognitiveProtocolBuilder` mapping, no confidence rule, no schema, no Angular behavior, no Tool Gateway behavior, and touches no MCP, pgvector, Azure Functions, or Kubernetes. It cites the vocabulary map (`protocol-vocabulary-map.md`) as the authoritative name registry and the lockstep rule.

The central finding: this is **not a flat rename.** Most fields rename one-to-one, but two changes are structural and need explicit approval (section 15): Stress relocates from Interrogate to Discern with a two-source collapse, and `perceive.detect`/`perceive.aim` are arrays today but scalars in the canonical `CognitiveProtocol` record.

## 1. current legacy field map

The analyzer emits one `phases` object. Grounded in `sports_analyzer.py` (prompt JSON lines 58-80, `_parse_phases` 330-349), the Pydantic models in `app/models/sports.py`, and the `.NET` mirror in `DevCore.AiClient/SportAnalysisContracts.cs`:

| protocol block | legacy field | type | prompt instruction |
|---|---|---|---|
| perceive | `detect` | `list[str]` | 1-3 attention triggers/anomalies |
| perceive | `frame` | `str?` | one-sentence factual context |
| perceive | `aim` | `list[str]` | 1-3 primary factors |
| interrogate | `balance` | `str?` | strongest case against the lean |
| interrogate | `stress` | `str?` | key risk / fragile assumption |
| interrogate | `reframe` | `str?` | alternate explanation |
| discern | `test` | `str?` | challenge to the emerging read |
| discern | `listen` | `str?` | market/external signal note |
| discern | `filter` | `str?` | evidence quality note |
| decide | `calibrate` | `str?` | why confidence matches evidence |
| decide | `posture` | `str?` | validated enum (play/pass/monitor/wait/compare/avoid) |
| decide | `voice` | `str?` | final framing, no hype |

Top-level response fields (not inside `phases`): `lean`, `lean_side`, `signals_used`, `summary`, `confidence`, `factors`, `posture` (duplicate of `decide.posture`), `counter_case` (= `interrogate.balance`), `watch_for` (= `interrogate.stress`), `what_would_change_the_read`.

Pydantic classes: `SportsCognitivePhases`, `SportsPerceivePhase`, `SportsInterrogatePhase`, `SportsDiscernPhase`, `SportsDecidePhase`. `.NET` mirror records of the same names in `SportAnalysisContracts.cs`, carried on `SportsAnalysisResponse.Phases`.

## 2. target canonical field map

The canonical vocabulary (from `protocol-vocabulary-map.md`, the `CognitiveProtocol` record, and the 15 station ids). The legacy-to-canonical mapping `CognitiveProtocolBuilder` performs today, which the migration moves up into the wire contract:

| canonical block | canonical field | legacy source today | rename class |
|---|---|---|---|
| perceive | `detect` | `perceive.detect[]` (joined "; ") | pure rename + array->scalar |
| perceive | `frame` | `perceive.frame` | pure rename (identical) |
| perceive | `aim` | `perceive.aim[]` (joined "; ") | pure rename + array->scalar |
| interrogate | `question` | `interrogate.balance` | pure rename |
| interrogate | `probe` | *(none -- deterministic `.NET` `BuildProbe`)* | not model-emitted |
| interrogate | `verify` | `interrogate.reframe` | pure rename |
| discern | `weigh` | `discern.filter` | pure rename |
| discern | `contrast` | `discern.listen` | pure rename |
| discern | `stress` | `interrogate.stress` ?? `discern.test` | **structural: relocation + two-source collapse** |
| decide | `resolve` | `decide.voice` | pure rename |
| decide | `position` | `decide.posture` | pure rename (value enum unchanged) |
| decide | `justify` | `decide.calibrate` | pure rename |
| synthesize | `integrate`/`compose`/`deliver` | *(none -- deterministic `.NET` constants)* | not model-emitted |

So eight pure renames (`balance->question`, `reframe->verify`, `filter->weigh`, `listen->contrast`, `voice->resolve`, `calibrate->justify`, plus `frame` and `posture->position` which keep value semantics), two array-to-scalar conversions (`detect`, `aim`), one structural relocation+collapse (Stress), and two fields the model never emits (Probe, Synthesize, both deterministic in `.NET`).

## 3. which FastAPI Pydantic models change

In `app/models/sports.py`:
- `SportsInterrogatePhase`: `balance -> question`, `reframe -> verify`, and `stress` is **removed from Interrogate** (moves to Discern). Class renamed `SportsInterrogateProtocol`.
- `SportsDiscernPhase`: `filter -> weigh`, `listen -> contrast`, `test` **removed**, `stress` **added** (the relocated/collapsed field). Class renamed `SportsDiscernProtocol`.
- `SportsDecidePhase`: `voice -> resolve`, `calibrate -> justify`, `posture -> position`. Class renamed `SportsDecideProtocol`.
- `SportsPerceivePhase`: `detect`/`aim` either stay `list[str]` (and `.NET` keeps joining) or become `str` (decision, section 15 Q2). `frame` unchanged. Class renamed `SportsPerceiveProtocol`.
- `SportsCognitivePhases` -> `SportsCognitiveProtocol`; the response member `phases` -> `protocol`.
- The model does **not** gain `interrogate.probe` or a `synthesize` block; those stay deterministic in `.NET`.

Compatibility option for the transition window: Pydantic `Field(validation_alias=AliasChoices("question","balance"))` (and equivalents) lets the parser accept either name from the model output while the prompt is updated, removing the alias once the prompt is canonical. Decision in section 15 Q4 (aliases vs hard-cut).

## 4. which prompt JSON instructions change

In `sports_analyzer.py` (the prompt `phases` skeleton at lines 58-80 and the per-field instructions at 118-168):
- rename the JSON keys in the skeleton and instructions: `balance->question`, `reframe->verify`, `filter->weigh`, `listen->contrast`, `voice->resolve`, `calibrate->justify`, `posture->position`, and the block label `phases->protocol`.
- **move** the `stress` instruction from the `interrogate` block to the `discern` block, and **delete** the separate `discern.test` instruction, folding its "challenge to the emerging read" intent into the single `discern.stress` instruction ("name the fragility / key risk / condition under which the read fails"). This is the prompt-content change with calibration impact (section 15 Q1).
- update the deliver-layer instruction lines (163-168) so `counter_case` is sourced from `interrogate.question` and `watch_for` from `discern.stress`. The product field names `counter_case` and `watch_for` themselves do **not** change (they are delivery extracts, not protocol fields).
- `_parse_phases` (`_parse_response`) updates its `.get()` keys to the canonical names, the relocated Stress source, and the renamed posture key (`position`), and the `counter_case`/`watch_for` extraction sources.
- `_validate_posture` is unchanged in behavior; only the key it reads (`position`) changes. The posture enum values are untouched.

## 5. which .NET AI client contracts change

In `DevCore.AiClient/SportAnalysisContracts.cs`, in lockstep with the Pydantic change (same wire shape):
- `SportsInterrogatePhase` -> `SportsInterrogateProtocol` (`Balance->Question`, `Reframe->Verify`, `Stress` removed).
- `SportsDiscernPhase` -> `SportsDiscernProtocol` (`Filter->Weigh`, `Listen->Contrast`, `Test` removed, `Stress` added).
- `SportsDecidePhase` -> `SportsDecideProtocol` (`Voice->Resolve`, `Calibrate->Justify`, `Posture->Position`).
- `SportsPerceivePhase` -> `SportsPerceiveProtocol` (`Detect`/`Aim` `string[]` or `string` per Q2).
- `SportsCognitivePhases` -> `SportsCognitiveProtocol`; `SportsAnalysisResponse.Phases` -> `Protocol` (JSON name `protocol`).
- `SportsAnalyzer.cs` consumes `response.Protocol` instead of `response.Phases`; the gateway-wrapped analyze call (`AnalysisSportsMatchupReadHandler`) is unchanged in transport, only the response member name changes.
- `counter_case`/`watch_for`/`posture`/`lean_side`/`signals_used` top-level `[JsonPropertyName]` mappings are unchanged.

## 6. which persisted artifact fields remain for compatibility

`OutputJson` today carries both legacy `CognitivePhases` and canonical `CognitiveProtocol` plus `ArtifactVersion` (`sports_decision_artifact_v2`). After migration:
- `CognitiveProtocol` stays the authoritative persisted block, now built near-directly from the canonical wire fields.
- introduce `ArtifactVersions.SportsDecisionArtifactV3 = "sports_decision_artifact_v3"` to mark canonical-native records (model emitted canonical names; no legacy translation occurred).
- **legacy `CognitivePhases` on new records:** recommend dropping it from v3 records (it was only a compatibility alias derived from the same single call). Old v1/v2 records keep their stored shape unchanged; nothing is rewritten. Decision in section 15 Q3 (drop on v3 vs keep deriving a legacy block for one more window).
- the read-side projection (`ProtocolView`) continues to: prefer `CognitiveProtocol` when present (v2/v3), fall back to legacy `CognitivePhases` for v1. No historical rewrite, ever (per the vocabulary map).

## 7. dual-emit from FastAPI during migration?

**Recommendation: no model-side dual-emit.** Do not make the model emit both a legacy `phases` block and a canonical `protocol` block. That doubles token cost and, worse, lets the two shapes diverge run-to-run (the model could phrase `balance` and `question` differently), defeating the point. The model emits exactly one deliberation block.

The safe "dual" lives at the boundary, not in the model:
- during the transition, the Pydantic/`.NET` parser accepts **either** name set via aliases (section 3, Q4), so a prompt rollout and a contract rollout do not have to be the same deploy.
- if a legacy-shaped persisted block is wanted for one window (Q3), `.NET` derives it deterministically from the canonical wire fields at compose time (the reverse of today's builder) -- a platform derivation, not a second model output.

This keeps "dual emit" as a persisted-artifact compatibility property (which already exists: `CognitiveProtocol` + `CognitivePhases`) and explicitly not a model-output property.

## 8. CognitiveProtocolBuilder: pass-through, compatibility mapper, or both?

**Both, with its job narrowed.** After migration `CognitiveProtocolBuilder`:
- becomes **pass-through** for the renamed scalar fields: `question`, `verify`, `weigh`, `contrast`, `resolve`, `position`, `justify`, `frame` arrive already canonical and are copied straight through.
- stays a **mapper** for the parts that are not model-emitted or not 1:1:
  - `interrogate.probe` -- still deterministic via `BuildProbe(followUps)`; unchanged.
  - `synthesize.{integrate,compose,deliver}` -- still the deterministic platform-operation constants; unchanged.
  - `perceive.detect`/`aim` -- still joins arrays to scalars if Q2 keeps arrays on the wire; becomes pass-through if Q2 moves to scalars.
  - `discern.stress` -- now sourced directly from canonical `discern.stress`; the old `interrogate.stress ?? discern.test` fallback is removed once the wire is canonical (the collapse happened in the prompt).
- keep the builder; do not delete it. It remains the single place that owns Probe, Synthesize, and the array handling. Renaming it is unnecessary; its responsibility shrinks, its name still fits.

The single-arg/two-arg `FromLegacy` overloads can keep their signatures for old-record reprocessing, or gain a `FromCanonical` sibling; decision deferred to the implementation slice (low risk, internal).

## 9. how /dev/artifacts handles v1 and v2 (and v3) records

No Angular behavior change is required by this migration. The `/dev/artifacts` page and the artifact endpoint already prefer persisted `CognitiveProtocol` and fall back to legacy `CognitivePhases` via the `ProtocolView` projection:
- v1 records (legacy only): unchanged legacy fallback path.
- v2 records (dual): unchanged canonical-preferred path.
- v3 records (canonical-native): read `CognitiveProtocol` directly -- already the preferred path; the page does not distinguish v2 from v3 in rendering. The Artifact Version stat simply shows `v3 sports_decision_artifact` for new runs.
- the page must continue to handle a v3 record whose legacy `CognitivePhases` is null (if Q3 drops it). The page already renders "Not recorded" when the legacy block is absent, so this is covered, but the implementation slice should confirm the v3-with-null-legacy case renders cleanly. The implementation slice keeps Angular code untouched unless that confirmation surfaces a gap.

## 10. how calibration scripts handle old and new outputs

`run-artifact-calibration.ps1` and `reconcile-calibration-outcomes.ps1`:
- read the artifact via the inspection endpoint and compute calibration flags primarily from `confidence` / `evidence_richness` / `signals_used` -- none of which are renamed by this migration. Those flags are unaffected.
- where a script reads phase-level text, it must read `CognitiveProtocol` (canonical) when present and fall back to legacy `CognitivePhases` for old records -- the same precedence the read-side projection uses. New v3 runs expose canonical names natively.
- historical calibration markdown and committed exports written under the legacy vocabulary are **not** rewritten (per the vocabulary map). Readers translate via the map.
- the implementation slice verifies the scripts against one v2 record and one v3 record before closing.

## 11. test plan

Lockstep, both sides, no behavior change to confidence or posture values:
- **FastAPI (pytest):** `_parse_phases` parses canonical keys; the relocated `discern.stress` is read from the discern block; `counter_case`/`watch_for` extraction reads `question`/`stress`; posture validation reads `position`; alias acceptance (if Q4 = aliases) parses both legacy and canonical keys during the window; malformed/missing fields stay null-safe.
- **.NET (xUnit):** `SportsAnalyzer` maps `response.Protocol` into the artifact; `CognitiveProtocolBuilder` pass-through preserves `question/verify/weigh/contrast/resolve/position/justify`; `Probe` and `Synthesize` still deterministic; `discern.stress` sourced from canonical; the v3 `ArtifactVersion` is stamped; `ProtocolView` projection still serves v1/v2/v3; the artifact endpoint contract for `/dev/artifacts` is unchanged at the JSON level for canonical consumers.
- **contract parity:** a round-trip test that the `.NET` `SportsAnalysisResponse` deserializes a canonical FastAPI payload (and, during the window, a legacy one via aliases).
- **regression:** the existing 248-test suite stays green; existing `CognitiveProtocolBuilderTests`, `ProtocolVocabularyMapperTests`, `SportsComposerTests`, `SportsAnalyzerTests` are updated in lockstep, not deleted.
- **ProtocolRegistry guard:** unaffected (station ids and tool ids do not change); the startup guard stays green.

## 12. rollback plan

- the migration ships behind the alias window (Q4): if the canonical prompt rollout misbehaves, revert the prompt commit and the parser still accepts the legacy names it emits, because aliases accept both. No data migration to undo.
- persisted records are never rewritten, so rollback is code-only: revert the prompt + Pydantic + `.NET` contract commits together (they are one lockstep change). v3 records already written remain readable because `CognitiveProtocol` is canonical regardless of which prompt produced it.
- if the structural Stress change (Q1) regresses calibration, it can be reverted independently of the pure renames if the implementation slice lands them as separate commits (section 13 sequences this).
- the Tool Gateway, confidence rules, and DB schema are untouched, so there is no stateful rollback surface.

## 13. exact implementation sequence

Each step is a small, separately revertible change. Renames first, structural changes last and isolated.

1. **Add aliases (additive, no behavior change).** Add canonical `validation_alias` choices to the Pydantic models and `[JsonPropertyName]`/alias handling to the `.NET` records so both name sets parse. Ship. The model still emits legacy names; nothing changes at runtime. This decouples the contract rollout from the prompt rollout.
2. **Pure renames in the prompt + parser.** Rename `balance/reframe/filter/listen/voice/calibrate/posture` to `question/verify/weigh/contrast/resolve/justify/position` in the prompt skeleton, instructions, and `_parse_phases`. Aliases keep the contract accepting them. Update the `.NET` builder pass-through and tests in lockstep.
3. **Rename classes and the `phases->protocol` member** across Pydantic and `.NET` in one commit (mechanical, compiler-checked on the `.NET` side).
4. **Structural Stress change (gated on Q1 approval).** Move the `stress` instruction to the discern block, delete `discern.test`, collapse to a single `discern.stress`. Update parser, builder Stress sourcing, and tests. Land as its own commit so it can revert independently. Run a calibration spot-check before and after.
5. **Array-to-scalar for detect/aim (gated on Q2).** If approved, change the wire type and drop the builder join; otherwise keep arrays and leave the join in the builder.
6. **Stamp `ArtifactVersion v3`** and decide legacy `CognitivePhases` persistence (Q3). Update `ProtocolView` and `/dev/artifacts` confirmation.
7. **Remove aliases** once the canonical prompt is the only producer and a calibration window has passed. This closes the migration; the contract is canonical-only.
8. **Update the vocabulary map and node-specs** to mark the legacy names as retired-from-runtime (still valid for reading historical records).

Steps 1-3 are low risk (renames). Steps 4-5 are the approval-gated structural changes. Step 7 is the final hard-cut.

## 14. what not to change in the implementation slice

- confidence values, bands, thresholds, or `SportsEvaluator` (the model never owned the number; this is a naming migration).
- the posture enum values (`play/pass/monitor/wait/compare/avoid`) or the `Read Stance` UI label.
- the deliver-layer product field names `counter_case`, `watch_for`, `what_would_change_the_read`, `lean`, `lean_side`, `summary`, `factors` (delivery extracts, not protocol fields).
- `interrogate.probe` determinism (`BuildProbe`) and the `synthesize` deterministic constants.
- the Tool Gateway, the tool ids, the station ids, or the startup guard.
- the DB schema; no migration, no historical rewrite.
- Angular rendering logic beyond confirming v3-with-null-legacy renders (and only if a gap is found).
- the retrieval clients, signal evaluators, or the analyze transport.

## 15. open questions requiring approval

1. **Stress collapse (highest impact).** Today the model emits two stress-adjacent sentences: `interrogate.stress` (key risk) and `discern.test` (challenge to the read). Canonical has one `discern.stress`. Collapsing two prompts into one changes what the model produces and may shift calibration. Approve: (a) collapse to one `discern.stress` now, (b) keep `discern.test` content mapped to a different canonical field, or (c) defer the collapse and only relocate `interrogate.stress` to `discern.stress` while temporarily keeping `discern.test` as a secondary source. Recommend (a) with a calibration spot-check, landed as its own revertible commit.
2. **detect/aim array vs scalar.** Canonical `CognitiveProtocol.Perceive.Detect/Aim` are scalars (joined today). Approve keeping `list[str]` on the wire (builder keeps joining) or moving to scalar at the prompt. Recommend keeping arrays on the wire (less prompt churn, the join is lossless and already tested); revisit later.
3. **Legacy CognitivePhases on v3 records.** Approve dropping the legacy block on v3 (rely on read-side projection for old records) or deriving it in `.NET` for one more window. Recommend dropping on v3; old records are untouched and still project.
4. **Alias window vs hard-cut.** Approve the alias-based transition (steps 1-7) or a single lockstep hard-cut deploy. Recommend the alias window: it decouples the prompt and contract rollouts and gives a clean rollback.
5. **`phases` -> `protocol` response member rename.** Approve renaming the response member (cleaner, canonical) or keeping `phases` as the JSON name with canonical fields inside (smaller diff). Recommend renaming to `protocol` during step 3 since it is compiler-checked on the `.NET` side and alias-guarded on the Python side.
6. **Artifact version constant.** Confirm `sports_decision_artifact_v3` as the canonical-native marker (parallel to the existing v2 constant).

## references

- `02 Platform/architecture/cognitive-factory/protocol-vocabulary-map.md` (authoritative name registry, lockstep rule)
- `02 Platform/architecture/cognitive-factory/protocol-station-blueprint-v1.md`
- `02 Platform/architecture/cognitive-factory/protocol-node-specs.md`
- `02 Platform/architecture/current-agent-run-contract.md` (persisted artifact, ArtifactVersion, ProtocolView)
- `02 Platform/architecture/current-sports-analysis-flow.md`
- code: `dai/services/agent-service/app/services/sports_analyzer.py` (prompt + `_parse_phases`), `app/models/sports.py`, `app/routes/sports.py`; `dai/platform/dotnet/DevCore.AiClient/SportAnalysisContracts.cs`; `dai/platform/dotnet/DevCore.Api/AgentRuns/CognitiveProtocol.cs`, `CognitiveProtocolBuilder.cs`, `SportsAnalyzer.cs`
