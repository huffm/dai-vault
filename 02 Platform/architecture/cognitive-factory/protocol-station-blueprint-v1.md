# Cognitive Protocol Station Blueprint v1

**date:** 2026-05-22
**status:** doctrine. defines the station-card contract for the Cognitive Protocol Runtime. no runtime code changed by this slice.
**scope:** how each cognitive node becomes a governed "station" with a compact, machine-loadable card, how the Tool Gateway derives permissions from those cards, and the migration path from vault doctrine to runtime config. docs only.

## what this document is

`cognitive-protocol-runtime.md` names the five macro protocols and their 15 micro-actions. `protocol-node-specs.md` gives each node an 11-facet operational spec in prose. `protocol-vocabulary-map.md` maps legacy code fields to canonical names. This document is the next layer: it defines a **station card**, a compact structured record per node that captures the same boundaries in a form the runtime can eventually load and the Tool Gateway can derive permissions from, without dumping doctrine prose into a model context window.

It is a blueprint, not an implementation. It introduces no runtime code. The station card is specified here so that a future `ProtocolRegistry v0` slice has an exact, stable target to encode against, exactly as `cloud-tool-runtime-plan.md` section 5 specified the `ToolDefinition` shape before the Tool Gateway was built.

Architecture direction this blueprint encodes (from the slice brief and existing doctrine):
- the artifact is shared state; protocols are stations; micro-actions are station-level jobs; workers execute bounded tasks.
- the Tool Gateway governs all tools. Scripts are deterministic reflexes. Vector memory is a future governed memory tool. Model calls are reserved for high-value cognition.
- Synthesize must not create new facts.
- the runtime must not dump the whole vault into model context; it should eventually load compact station cards or protocol config.

Guardrails preserved: this slice changes no FastAPI prompt, no Pydantic contract name, no `CognitiveProtocolBuilder` mapping, no confidence rule, no database schema, no Angular behavior, and touches no MCP, pgvector, Azure Functions, Kubernetes, secret, or customer-auth implementation. The Cognitive Protocol Runtime behavior, Tool Gateway governance, and `CognitiveProtocol` persistence are unchanged. Nothing here bypasses the Tool Gateway.

## 1. what a protocol station is

A **protocol station** is the runtime, operational embodiment of one canonical micro-action node. "Node" is the doctrine word (`protocol-node-specs.md`); "station" is the same thing seen from the factory framing in `CLAUDE.md` (platform = factory, niches = assembly lines): a fixed position on the line where a bounded job is done on the shared decision artifact as it passes through.

A station:
- occupies one position in the fixed pipeline Perceive -> Interrogate -> Discern -> Decide -> Synthesize.
- reads a scoped set of artifact fields, does one bounded job, and writes one scoped output field.
- has a closed list of tools, scripts, model calls, and (future) memory queries it may invoke. It cannot widen its own permissions.
- fires on specific inputs and follows a fixed schema; it does not roam, choose its own inputs, or invent new facts.

There are 15 stations, one per node: Perceive {Detect, Frame, Aim}, Interrogate {Question, Probe, Verify}, Discern {Weigh, Contrast, Stress}, Decide {Resolve, Position, Justify}, Synthesize {Integrate, Compose, Deliver}. Twelve are cognitive micro-actions (Perceive + Interrogate + Discern + Decide); the Synthesize trio is platform-operational and not counted among the twelve. Of the twelve cognitive stations, **eleven are model-emitted** (produced inside one analyze call as the `SportsAnalyzerProtocolSeed`) and **exactly one — Interrogate.Probe — is deterministic** platform output completed by `CognitiveProtocolBuilder.BuildProbe`. So the split is 11 model-emitted + 4 deterministic (Probe plus the Synthesize trio) = 15. Interrogate is a three-station macro even though the model emits only two of its stations. Retrieve is **not** a station; it is platform plumbing that runs before Perceive and stamps grounded evidence onto the artifact.

A station is not an agent. It does not plan, it does not pick its tools, and it does not decide what comes next. The pipeline order is platform code; the permissions are the card; the cognition is bounded text filling pre-scoped fields.

## 2. what a station card is

A **station card** is a compact structured record (target encoding: one entry in a static manifest, mirroring `ToolRegistry.Default()`) that declares everything the runtime needs to run a station safely: its identity, what it reads and writes, what it may invoke, its contracts, its quality gates, its fallback, its forbidden behavior, and its telemetry and calibration hooks.

The card is deliberately small. It is the unit the runtime loads instead of loading doctrine prose. A model call for a cognitive station receives only the scoped inputs plus, where useful, the card's purpose and forbidden-behavior lines as bounded instruction; it never receives `protocol-node-specs.md` wholesale. The card is the compaction boundary that keeps the runtime from dumping the vault into context.

A station card is read-only doctrine-derived config. It is not a place for niche business logic, prompts, or scoring thresholds; those live in retrievers, prompt bodies, posture vocabularies, and `SportsEvaluator`. The card carries permission, contract, and boundary, the same division the Tool Gateway already enforces for tools.

## 3. how station cards relate to protocol-node-specs.md

`protocol-node-specs.md` stays the human-readable doctrine: 15 nodes, 11 prose facets each, sports overlays, open questions. The station card is the **machine-readable projection** of that doctrine, plus the runtime/governance metadata the prose does not carry.

Mapping from the node-specs 11 facets to station-card fields:

| node-specs facet | station-card field(s) |
|---|---|
| Purpose | `purpose` |
| Artifact fields read | `artifact_fields_read` |
| Artifact fields written | `artifact_fields_written` |
| Allowed tools | `allowed_tools` |
| Allowed scripts / reflexes | `allowed_scripts` |
| Model-call rule | `allowed_model_calls` |
| Signal dependencies | folded into `input_contract` |
| Quality checks | `quality_gates` |
| Fallback behavior | `fallback_behavior` |
| Forbidden behavior | `forbidden_behavior` |
| Sports overlay | competition mapping (section 12), not a card field |

Station cards add fields the prose does not have: `station_id`, `macro_protocol`, `micro_action`, `output_contract`, `allowed_memory_queries`, `token_budget`, `cost_class`, `telemetry_tags`, `calibration_hooks`. These come from `cognitive-protocol-runtime.md` ("what a protocol owns" already lists token budget and calibration hooks) and from the `ToolDefinition` metadata model.

Rule: **node-specs is the source; the card never contradicts it.** When they ever diverge, node-specs wins and the card is corrected. A future `ProtocolRegistry v0` should be generated from or validated against node-specs, not hand-drifted.

## 4. how the runtime moves from vault doctrine to runtime config

Today: doctrine lives in the vault (`protocol-node-specs.md` and friends). The runtime does not read it. The 12 cognitive micro-actions are produced by one analyze prompt in FastAPI whose body encodes the boundaries informally; the deterministic stations (Probe, Synthesize) encode them in `CognitiveProtocolBuilder` and `SportsComposer`. The Tool Gateway enforces tool permission at the platform-stage level (`platform.reference`, `platform.retrieve`, `platform.analyze`).

Target: the boundaries become a small, versioned, machine-loadable manifest of station cards (`ProtocolRegistry v0`), living in code next to `ToolRegistry`, the same way the tool manifest does. The runtime loads cards, not prose. A station's model call is built from its card's scoped inputs and bounded instruction lines; the Tool Gateway checks a station's tool invocation against the card-derived permission set (section 6).

The move is staged, never a big-bang rewrite (section 13). The card contract is fixed first (this doc), the manifest is encoded second (`ProtocolRegistry v0`, declarative and unenforced like the early `ToolDefinition` metadata), enforcement and per-station model construction follow only when the FastAPI canonical-field migration makes per-station encoding honest. Until then doctrine and runtime stay reconciled by hand through node-specs and the vocabulary map.

## 5. required station card fields

Every station card carries exactly these fields. Empty is written as an explicit empty list or `none`, never omitted, so the shape is trusted.

| field | type | meaning | grounded in |
|---|---|---|---|
| `station_id` | string | the canonical node id, e.g. `interrogate.probe`, `decide.position`. **Reuses the existing canonical node-id string**, the same value used as `ToolInvocationContext.ProtocolNode`. No new id scheme. | `ToolInvocationContext`, vocabulary map |
| `macro_protocol` | enum | one of `perceive`, `interrogate`, `discern`, `decide`, `synthesize` | runtime doc |
| `micro_action` | string | the micro-action name, e.g. `probe`, `position` | runtime doc |
| `purpose` | string | the single responsibility in one or two sentences | node-specs facet 1 |
| `artifact_fields_read` | list | scoped inputs on the shared artifact; the station reads nothing else | node-specs facet 2 |
| `artifact_fields_written` | list | the scoped output field(s); the station writes nothing else | node-specs facet 3 |
| `allowed_tools` | list of tool_id | the closed list of Tool Gateway tool ids this station may invoke. Empty for stations that make no tool call (most cognitive stations today invoke no tool directly; the analyze call is made by the platform stage). | node-specs facet 4, `ToolRegistry` |
| `allowed_scripts` | list | deterministic reflexes that run for this station with no model, e.g. `CognitiveProtocolBuilder.BuildProbe`, `SignalQualityEvaluator` | node-specs facet 5 |
| `allowed_model_calls` | enum | `none` (deterministic station) or `shared_analyze` (content emitted inside the single analyze call) or, post-migration, a specific analyze tool id. Encodes the model-call rule. | node-specs facet 6 |
| `allowed_memory_queries` | list of memory tool_id | governed vector-memory queries this station may run. **Empty for every station at launch** (section 10). | section 10, cloud plan section 8 |
| `input_contract` | typed shape | the required input shape and signal dependencies; what must be present for the station to fire | node-specs facets 2 and 7 |
| `output_contract` | typed shape | the exact written shape, e.g. scalar string, enum, string list; clamps where applicable (e.g. Position is the closed posture enum) | node-specs facet 3, agent-run contract |
| `quality_gates` | list of gate_id | the deterministic checks and calibration flags that police this station, by id (e.g. `counter_case_generic`, `confidence_high_for_partial_evidence`) | node-specs facet 8, section 11 |
| `fallback_behavior` | string | what the station does when inputs are missing or degraded | node-specs facet 9 |
| `forbidden_behavior` | list | what the station must never do | node-specs facet 11 |
| `token_budget` | int or null | the bound on model tokens when the station calls a model; null for deterministic stations | runtime doc ("token budget when the protocol calls a model") |
| `cost_class` | enum | `free`, `cheap`, `paid_external`, `model_call`, mirroring `ToolCostClass` | `ToolDefinition` |
| `telemetry_tags` | list | the structured-log dimensions emitted when this station runs; aligns with `ToolGatewayInvocation` (tool id, protocol node, run id, tenant key, correlation id, cost class, outcome, duration) | `ToolTelemetry`, cloud plan section 8 |
| `calibration_hooks` | list of metric_id | the calibration metrics this station feeds or is measured by | runtime doc, section 11 |

`station_id` reusing the canonical node id is the load-bearing naming decision: it keeps one keyspace across the vocabulary map, `ToolInvocationContext.ProtocolNode`, the Tool Gateway `AllowedProtocolNodes` sets, and the station cards. A second id scheme would fork that keyspace and is rejected.

## 6. how Tool Gateway permissions are derived from station cards

Today the relationship runs tool -> nodes: each `ToolDefinition` carries an `AllowedProtocolNodes` set, and `ToolGateway.InvokeAsync` fails closed unless `context.ProtocolNode` is in that set. The station card runs the inverse direction node -> tools: a card's `allowed_tools` lists the tool ids that station may invoke.

The two are duals of one permission matrix. The intended derivation:

- a station may invoke a tool **iff** the tool id is in the station's `allowed_tools` **and** the station's `station_id` is in that tool's `AllowedProtocolNodes`. Both directions must agree. This is a cross-check, not a second permission system.
- a future `ProtocolRegistry v0` lets the gateway (or a startup validator) confirm the two manifests are consistent: every `allowed_tools` entry on a card resolves to a real tool whose `AllowedProtocolNodes` contains that `station_id`, and no tool names a `station_id` that the corresponding card omits. A mismatch fails the build, not a run.
- nothing about the gateway interface changes. The gateway still enforces `AllowedProtocolNodes`; the station card is the authoritative source the tool's allowed-nodes set should be generated from or validated against, replacing today's hand-maintained sets.

Important honesty about today: the 12 cognitive micro-actions are produced inside one analyze call whose gateway caller is the platform stage `platform.analyze`, not a per-node id. So the gateway currently enforces at the **stage** level (`platform.reference`, `platform.retrieve`, `platform.analyze`), and the per-station `allowed_tools` lists are doctrine that becomes enforceable only when stations make their own gateway calls (post FastAPI canonical migration). The card is written now so that enforcement is a configuration step later, not a redesign.

## 7. how scripts and reflexes fit without becoming agents

A **script (reflex)** is deterministic code that runs for a station with no model call. Examples already in the runtime: `SignalQualityEvaluator` and `SignalFollowUpEvaluator` (retrieval-side grading), `CognitiveProtocolBuilder.BuildProbe` (the Probe station), `SportsComposer` and `CognitiveProtocolBuilder.FromLegacy` (the Synthesize trio), `SportsEvaluator` (the confidence number and band), the `LeanSide` and posture-enum clamps.

Reflexes stay non-agentic because:
- they are listed in the card's `allowed_scripts` and run only for their station. A reflex does not choose when to fire; the pipeline position does.
- they are pure or near-pure deterministic computation with a fixed output shape. They add no new facts (Probe drops any signal lacking a doctrinal template rather than inventing one; the Synthesize trio plates only what survived).
- they own numbers and enums where determinism is honest. `SportsEvaluator` owns `Confidence`/`ConfidenceBand`; the model never overrides them. Posture is clamped to the closed enum.
- a reflex never calls a model and never widens permissions. `allowed_model_calls = none` on every reflex-only station.

The rule: where determinism is honest, a reflex does the job; the model is not invited. A reflex is a tool of the station, not an actor.

## 8. how model calls fit without becoming freeform agents

A **model call** is reserved for cognition that genuinely benefits from interpretation: Question and Verify (Interrogate), Contrast and Stress (Discern), Resolve and Justify (Decide), and today Detect/Frame/Aim (Perceive). The model fills bounded text into pre-scoped fields. It is not an agent because:

- the model never selects tools, never widens data access, and never writes outside its station's `artifact_fields_written`. Permissions are the card, not the model's choice.
- the model owns judgment text only. It does not own numbers (confidence is `SportsEvaluator`), does not own the posture value (clamped enum), and does not own artifact shape (deterministic Synthesize).
- the call is bounded by the card's `token_budget` and `input_contract`. It receives scoped inputs, not the vault.
- output is policed by `quality_gates` (prohibited-phrase guardrails, calibration flags) after it runs.
- it is one analyze call today, not 12. Splitting into per-station calls or skill packs is an undecided future option (`cognitive-skill-pack-architecture-v1.md`); the per-station contracts hold either way.

The rule: model calls are a high-value cognition budget, spent only where interpretation earns it, inside a fixed schema, never as freeform autonomy.

## 9. document analysis as governed tools, not unrestricted context loading

The runtime must never load whole markdown or HTML documents (the vault, a niche knowledge pack, a scraped page) into a model context window. Document analysis is represented as **governed document tools** routed through the Tool Gateway, exactly like every other tool, so the model receives a bounded, scoped extract rather than raw text.

Shape (future tools, not built this slice; dotted namespace consistent with existing tool ids):
- `document.markdown.extract` and `document.html.extract`: take a document reference plus a scoped query and return a bounded, structured extract (the relevant section or fields), never the full document.
- `ToolKind = Retriever` (or a future `Evaluator` when the tool also grades), `Transport = InProcNet` or `HttpExternal` depending on where the document lives, `CostClass = cheap`, `Idempotency = PerRunInput`.
- `AllowedProtocolNodes`: empty at launch. A document tool opens to a station only when a node spec and a calibration finding justify it (the first realistic opening is Interrogate.Probe, parallel to memory in section 10).
- the extract is bounded by the calling station's `token_budget`; the gateway logs the invocation like any tool.

This keeps "load compact station cards or protocol config" and "do not dump the vault into context" as the same discipline: doctrine and documents both enter cognition only as governed, bounded, gateway-logged tool outputs.

## 10. how vector memory fits later

Vector memory (pgvector, deferred per `cloud-tool-runtime-plan.md` section 8) enters as **governed memory tools** through the Tool Gateway. It is additive memory and search, never a primary store, and never a context dump. The memory tool ids:

| memory tool_id | purpose | first plausible station |
|---|---|---|
| `memory.prior_runs.search` | find prior runs with a similar matchup or signal shape | Interrogate.Probe |
| `memory.calibration.lessons` | retrieve calibration lessons relevant to this matchup or signal set | Discern.Weigh / Decide.Justify |
| `memory.niche.rules` | retrieve niche-specific rules for the competition or product | Perceive.Aim / Decide.Position |
| `memory.signal_failures` | retrieve known signal-failure patterns for the grounded/missing signal set | Interrogate.Probe / Discern.Stress |

Rules:
- each is a `ToolDefinition` with `ToolKind = Retriever`, `CostClass = cheap`, an idempotency rule, and a `secrets_scope` for the Postgres connection. The handler is a typed memory client inside .NET; the station does not learn the transport.
- `allowed_memory_queries` is **empty on every station card at launch.** `AllowedProtocolNodes` on each memory tool is empty for synchronous use and is opened to exactly one station only when a calibration finding supports it. The first opening is `memory.prior_runs.search` and `memory.signal_failures` to `interrogate.probe`, where prior-run lookups inform investigation candidates.
- memory returns bounded extracts, governed and logged like any tool. It never replaces the deterministic Probe templates by default; it augments behind a per-tenant flag.
- offline first: calibration-side ingestion and a read-only review surface land before any synchronous station opens against memory.

## 11. how ML and calibration fit later

The learning loop is already half-built in code and stays a closed, governed loop, not an autonomy layer:

1. **outcomes feed calibration.** `AgentRunOutcome` records the raw real-world result; `RunEvaluator` derives `AgentRunEvaluation` (`correct`/`incorrect`/`inconclusive`) in the same transaction. Offline, `run-artifact-calibration.ps1` and `reconcile-calibration-outcomes.ps1` join runs to outcomes and compute calibration flags (`confidence_high_for_partial_evidence`, `counter_case_generic`, `frame_missing_rest_context`, and so on).
2. **calibration informs quality gates.** Calibration flags are distinct from `ArtifactQualityWarnings` (the deterministic `SportsQualityChecker` surface stored on the artifact). A station card's `quality_gates` references both kinds by id. As calibration accumulates evidence, gates are tightened, added, or retired in doctrine first, then in code.
3. **quality gates may eventually adjust confidence or trigger critique.** When enough reconciled game-days exist, a gate may feed back into `SportsEvaluator`'s calibration tier (adjusting confidence) or trigger a critique station (a bounded re-examination). Today this is deferred: confidence calibration stays manual until there is enough reconciled evidence (the handoff records the standing decision that a handful of leans is too small to move a threshold). No confidence rule is changed by this blueprint.

The card's `calibration_hooks` field names the metrics each station feeds or is measured by, so the loop knows which station a flag implicates without re-deriving it.

## 12. sports v1 station mapping

The station contract is identical across competitions; what differs per competition is the grounded signal set and therefore which stations have rich inputs versus thin ones. From the sports signal map in `protocol-node-specs.md`:

| competition | code | analyzer family | grounded signals | station consequences |
|---|---|---|---|---|
| NBA | `nba` | basketball | `rest_schedule`, `market`; `sharp_public` attempted (null in playoffs) | Contrast has market-vs-rest and market-vs-sharp/public when present; Probe's `sharp_public` template fires often; Position respects `block_aggressive_posture` when sharp/public missing |
| NCAAM | `ncaamb` | basketball | `rest_schedule`, `market` | as NBA but no season-long injury/availability source; do not force injury grounding; `sharp_public` typically absent |
| NFL | `nfl` | football | `market`; `sharp_public` attempted (regular season) | Contrast is market-vs-sharp/public; Stress often names spread-cover risk or market-only lean |
| NCAAF | `ncaaf` | football | `market` | shares the football family; station contracts apply identically to NFL |
| MLB | `mlb` | mlb | `starting_pitching` only | single-signal runs: `EvidenceRichness = 1`, Contrast usually limited or absent, Probe usually null (fires only if a starter is unavailable); Justify uses single-signal calibration |
| NCAAW | n/a | n/a | n/a | **not a platform competition.** No `CompetitionCatalog` entry, no seeded teams, no retriever route. No station overlay can be written. See note. |

**NCAAW note (carried from node-specs open question 1):** women's college basketball is out of scope. The blueprint does not invent a station overlay for it. Adding NCAAW is a separate slice (add to `CompetitionCatalog`, seed teams, add a retriever route) and must precede any cognitive mapping. This blueprint flags the gap and stops, rather than fabricating signals or overlays the platform cannot ground.

## 13. near-term migration path

Sequenced and small, each step shippable, no big-bang rewrite:

1. **keep the current dual emit.** New runs persist both legacy `CognitivePhases` and canonical `CognitiveProtocol`; old runs deserialize via the legacy fallback. No schema change, no historical rewrite. (In place today.)
2. **keep the Tool Gateway.** Reference, retrieve, and analyze all route through the gateway with `ToolGatewayInvocation` telemetry. (In place today.)
3. **add the station blueprint.** This document fixes the station-card contract. Docs only. (This slice.)
4. **later: add runtime `ProtocolRegistry v0`.** Encode the 15 station cards as a static manifest in code next to `ToolRegistry`, declarative and unenforced first (like the early `ToolDefinition` metadata). Add a startup validator that cross-checks card `allowed_tools` against tool `AllowedProtocolNodes` (section 6). No behavior change; this is the machine-readable encoding of node-specs.
5. **later: migrate FastAPI to canonical fields.** Update the analyze prompt to emit canonical micro-action names and rename the Pydantic models and the .NET records in lockstep (per the vocabulary map's lockstep rule). This is what makes per-station model construction and per-station gateway enforcement honest. Confidence rules, posture enum, and the deterministic stations are untouched by the rename.
6. **later: add vector memory as a governed tool.** Provision pgvector offline-first, build calibration-side ingestion and a read-only review surface, then open `memory.prior_runs.search` / `memory.signal_failures` to `interrogate.probe` behind a per-tenant flag (section 10).

Each "later" step is gated on the one before it and on calibration evidence where confidence or memory is involved.

## 14. launch-required vs post-launch

**Launch-required (for the sports v1 launch on Container Apps):**
- the Cognitive Protocol Runtime behavior, dual emit, and `CognitiveProtocol` persistence exactly as they are today.
- the Tool Gateway governing reference, retrieve, and analyze with telemetry, exactly as today.
- this station blueprint as the fixed contract (docs).
- everything in the Azure Container Apps Provisioning Plan v1 and Customer Auth Readiness v1 that gates public exposure (those are separate slices; this blueprint does not add launch infrastructure).

**Post-launch (deferred, in order of the migration path):**
- runtime `ProtocolRegistry v0` (the machine-readable card manifest plus the consistency validator).
- the FastAPI canonical-field migration that enables per-station model construction and per-station gateway enforcement.
- vector memory tools and the document tools in sections 9 and 10.
- calibration feeding back into confidence or triggering a critique station (section 11), gated on reconciled-outcome evidence.
- splitting the single analyze call into per-station calls or skill packs (undecided; not required for launch).

The blueprint is launch-neutral: it adds doctrine, not runtime surface, so it does not move the launch gate.

## 15. what not to build yet

Called out so they cannot quietly become scope creep:
- **no `ProtocolRegistry` code this slice.** The card contract is fixed in docs; encoding it is the next slice, with its own tests. A code stub now would be untested runtime surface for no behavior gain.
- **no per-station model calls.** One analyze call stays one analyze call until the FastAPI canonical migration; do not split prompts or routes for stations now.
- **no vector memory, no document tools wired.** `allowed_memory_queries` stays empty on every card; `memory.*` and `document.*` tools are specified, not registered. No pgvector provisioning here.
- **no confidence-rule change, no posture-enum change, no calibration auto-feedback.** Confidence stays deterministic and manually calibrated; gates only flag.
- **no FastAPI prompt, Pydantic, `CognitiveProtocolBuilder`, schema, or Angular change.** The vocabulary-map lockstep rule governs the eventual rename; it is not this slice.
- **no NCAAW overlay.** Out of scope until the competition exists.
- **no MCP, Azure Functions, AKS, or multi-region work.** Deferred per `cloud-tool-runtime-plan.md`.

## references

- `02 Platform/architecture/cognitive-factory/cognitive-protocol-runtime.md`
- `02 Platform/architecture/cognitive-factory/protocol-node-specs.md`
- `02 Platform/architecture/cognitive-factory/protocol-vocabulary-map.md`
- `02 Platform/architecture/cognitive-factory/cognitive-skill-pack-architecture-v1.md`
- `02 Platform/architecture/cloud-tool-runtime-plan.md` (sections 4-8: Tool Gateway, registry, Functions, pgvector)
- `02 Platform/architecture/current-agent-run-contract.md` (artifact contract, outcome/evaluation entities)
- `02 Platform/architecture/current-sports-analysis-flow.md`
- code: `dai/platform/dotnet/DevCore.Api/Tools/` (`ToolDefinition.cs`, `ToolInvocationContext.cs`, `ToolRegistry.cs`, `ToolGateway.cs`, `ToolTelemetry.cs`), `DevCore.Api/AgentRuns/` (`CognitiveProtocolBuilder.cs`, `SportsComposer.cs`, `SportsEvaluator.cs`)
