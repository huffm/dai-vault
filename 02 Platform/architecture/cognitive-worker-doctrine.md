# cognitive worker doctrine

**date:** 2026-05-01
**status:** foundational platform doctrine — applies to all niche assembly lines

---

## core claim

DAI does not treat AI as a magic answer generator.
DAI treats AI as a bounded cognitive worker inside a structured artifact manufacturing process.

The platform maximizes AI cognition by controlling:
- **context** — only grounded, relevant material enters the cognitive process
- **role** — the worker knows what it owns and what it does not
- **boundary** — the worker knows what it must not claim or invent
- **memory** — prior artifacts, outcomes, and known facts are staged appropriately
- **evaluation** — the platform scores, calibrates, and validates output before it reaches a user

Myth inspires the worker.
Logic defines the job.
The artifact proves the value.

The symbolic anatomy of the cognitive pipeline is design lineage only.
The software implementation must remain grounded, logical, and operational.

---

## maximizing AI cognition

The key constraint: we rely on third-party models. We do not control weights, fine-tune, or retrain.

Cognition is maximized not by improving the model, but by improving the conditions under which it works.

Control what enters the context. Stage material clearly. Define the role precisely. Bound what may be claimed. Validate what comes out. Track accuracy over time.

A well-staged prompt with grounded signals, explicit boundary rules, and a structured output schema will consistently outperform the same model given a raw question with no structure.

---

## AI as a structured participant

The model participates inside the artifact pipeline as a cognitive worker.
It does not own the pipeline. It does not decide what to retrieve, what to score, or what to store.
It is given a scoped task, well-staged inputs, and a defined output contract.

Its value is in:
- interpreting ambiguous signals
- generating counter-cases and stress-testing the read
- recognizing patterns across grounded facts and context
- expressing calibrated uncertainty honestly
- synthesizing validated material into coherent, truthful language

Its risk is in:
- inventing facts not in the context
- fabricating certainty not earned by evidence
- obscuring uncertainty to sound more confident
- overriding evidence quality with fluency

The platform's job is to reduce the risk and maximize the value.

---

## platform responsibilities

The platform owns:
- tenant boundaries and access scoping
- retrieval — what gets fetched, from where, and how it is staged
- schema validation — what shapes are allowed in and out
- enum clamping — invalid or out-of-vocabulary values become null before they propagate
- evidence grounding — knowing which signals have real retrieved data behind them
- confidence calibration — deterministic rules based on evidence richness, not model self-assessment
- persistence — artifact storage, audit trail, run history
- evaluation — outcome tracking, correct/incorrect/inconclusive judgment
- tool routing and API orchestration

The platform does not ask the model to perform any of these.

---

## AI responsibilities

The model owns:
- interpretation of ambiguous or conflicting signals
- counter-case generation — the strongest argument against the lean
- language judgment — clarity, calibration, no hype
- pattern recognition from grounded and contextual material
- uncertainty expression — what the model does not know is as important as what it knows
- synthesis of validated material into the final consumable artifact

The model does not own:
- evidence richness (that is the platform's grounded signal count from the retriever)
- final confidence (the platform calibrates this deterministically after the model call)
- fact invention (the model may only reference what is in the context or accepted interpretation)
- posture enforcement (posture is validated and clamped by the platform before reaching the user)

---

## deterministic code responsibilities

Code owns:
- schema definitions and enforcement
- tenant and user scoping
- retrieval orchestration
- SQL queries and structured state retrieval
- vector search orchestration (when present)
- validation and enum clamping
- persistence and audit trail
- confidence and evidence scoring where rule-based
- outcome tracking and evaluation judgment
- calculations (spreads, rest days, days since last game, evidence richness)

Code does not ask the model to perform any of these.

---

## memory and retrieval

**SQL / Postgres:** stores durable truth — run history, tenants, teams, outcomes, prompt versions, structured facts.
JSONB columns store evolving artifact shapes without requiring a migration for every new field.
SQL retrieves trustworthy structured state.

**Vector search / pgvector (future):** acts as associative memory for prior artifacts, similar cases, notes, and unstructured context. a complement to SQL for semantic retrieval, not a replacement for it.

None of these are "the brain." They become useful when staged into the cognitive artifact — as grounded inputs the model can reason against, not as raw dumps it must interpret unaided.

Do not build vector infrastructure until there is a specific retrieval use case that structured SQL cannot serve.

---

## known facts vs AI interpretations

The artifact must clearly distinguish:

| category | examples |
|---|---|
| known facts | game date, team names, spread value, days rest, start time |
| retrieved signals | current spread, sharp/public split, probable starters, weather |
| calculated values | days since last game, evidence richness count, calibrated confidence |
| AI interpretations | lean direction, counter-case, posture rationale, confidence narrative |
| missing information | signals not retrieved, starters not announced, no weather data |
| excluded inputs | signals explicitly out of scope for this run type |
| uncertainty | where the model does not have grounded evidence to reason from |

Known facts must not be mixed with AI interpretation.
Calculated values must not be attributed to model output.
Missing information must be surfaced, not silently omitted.

---

## augmentation vs invention

**Augmentation** means transforming, organizing, challenging, comparing, compressing, clarifying, or contextualizing material already present in the artifact.

Augmentation is what the model should do.

**Invention** means adding unsupported claims, fabricated facts, fake certainty, or conclusions not grounded in known facts, retrieved signals, or accepted interpretations.

Invention is what the model must be prevented from doing.

The platform's prompt boundaries, schema constraints, and posture vocabulary are the primary tools for preventing invention.

---

## cognitive phase ownership

### perceive

**owns:**
- receiving the context field (competition, teams, date, grounded signals)
- detecting attention triggers and anomalies
- framing the factual matchup context
- aiming attention at the factors that matter most for this decision

**must not:**
- decide final posture
- invent missing data
- overstate signal strength

---

### interrogate

**owns:**
- applying pressure to the initial read
- balance — stating the strongest case against the lean
- stress — surfacing the fragile assumption or key risk
- reframe — offering alternate explanations or non-obvious angles

**must not:**
- add new facts not in the context
- attack for the sake of noise or word count
- decide final stance

---

### discern

**owns:**
- judging evidence quality
- listen — interpreting external signals (market, sharp/public) when present
- filter — separating grounded evidence from weak or absent evidence
- distinguishing what is known from what is inferred

**must not:**
- treat model interpretation as grounded fact
- admit stale or weak evidence as equivalent to strong grounded evidence
- ignore missing information

---

### decide

**owns:**
- calibrate — explaining why the confidence level fits the evidence
- posture — the read stance from the allowed vocabulary
- voice — the final framing without hype or false certainty

**must not:**
- let confidence exceed what the evidence supports
- use hype or lock language
- call uncertain reads certain
- produce a posture not supported by the evidence quality

---

### synthesize

**owns:**
- integrate — combining validated material from prior phases without adding new claims
- compose — assembling the final decision artifact shape
- deliver — presenting the consumable artifact clearly and honestly

**must not:**
- invent new evidence
- introduce unsupported arguments or claims
- hide uncertainty surfaced in prior phases
- override evidence quality with more confident language

**conceptual → code → user-facing mapping:**

| layer | name |
|---|---|
| vault / conceptual | Manifest |
| software / doctrine | Synthesize |
| pipeline stage code | Compose |
| user-facing output | Deliver |

Synthesize integrates, compresses, augments, and presents validated artifact material.
It does not invent. It does not add unsupported claims. It does not override evidence quality.
It produces the consumable decision artifact from what survived the prior phases.

---

## shared artifact template

This is target doctrine — the conceptual contract for the full decision artifact.
It does not exactly match current code. Current code stores a subset of these categories in `OutputJson`.

```json
{
  "decision_problem": {},
  "known_facts": [],
  "retrieved_signals": [],
  "calculated_values": [],
  "ai_interpretations": [],
  "missing_information": [],
  "excluded_inputs": [],
  "uncertainty": [],
  "perceive": {},
  "interrogate": {},
  "discern": {},
  "decide": {},
  "synthesize": {},
  "final_artifact": {}
}
```

The current `OutputJson` carries `CognitivePhases` (perceive, interrogate, discern, decide) and compact deliver fields. The full template above is the direction — not yet implemented.

---

## worker implementation types

Workers are responsibilities before they are services.

A worker may be implemented as:
- deterministic code
- SQL query
- vector retrieval
- tool call
- a prompt section inside a single model call
- a model call
- a schema validator
- a scoring rule
- a future specialized sub-agent

Do not imply every worker must become a service.
Do not build separate model calls until a single well-structured prompt fails to satisfy the need.
Do not build a workflow engine to express structure that a prompt schema can express.

---

## when more AI calls are justified

**v1:** one model call with structured phase outputs is acceptable.
The structured prompt guides the model through perceive → interrogate → discern → decide.
The platform handles synthesize via deterministic code in `SportsComposer`.

**later:** an optional additional model call may be added when the artifact genuinely requires it:
- confidence is high but evidence richness is low (model may be overconfident)
- posture is aggressive but stress notes are severe
- signals conflict significantly
- counter-case is missing or weak
- what-would-change-the-read is absent
- the final artifact fails a quality validation rule

Do not add a second model call because it feels more sophisticated.
Add it only when artifact quality measurably improves as a result.

---

## measuring worker value

Each phase must earn its place. Measurement is not optional.

| phase | what it must improve |
|---|---|
| perceive | context completeness — grounded signals surfaced correctly |
| interrogate | counter-case quality — the lean holds up when challenged |
| discern | evidence hygiene — weak signals correctly filtered or flagged |
| decide | calibration accuracy — stated confidence tracks actual lean correctness |
| synthesize | clarity, trust, and usability — the artifact helps users make better decisions |

Outcome measurement should include:
- lean direction correctness rate by competition and signal tier
- calibration quality (stated confidence vs outcome frequency over time)
- artifact completeness (are counter-case, watch-for, and change conditions present?)
- user trust and comprehension
- repeat usage over time
- whether the read helped a user avoid a bad action, not just confirm a good one

---

## generalizing beyond sports

Sports is the first niche assembly line. The cognitive worker process is the reusable platform layer.

The same pipeline — context control, role definition, boundary enforcement, evidence grounding, structured output, synthesis — should generalize to:
- crypto market reads
- stock entry or exit evaluation
- job lead scoring
- vendor or tool comparison
- grant opportunity triage
- business decision memos
- compliance or risk triage
- any decision where evidence exists, uncertainty is real, and the user has something at stake

**the platform owns:** the reusable cognitive pipeline, artifact shape, storage, evaluation, and calibration layer.

**the niche owns:** data sources, retriever implementations, prompt families, posture vocabulary, scoring rules, confidence thresholds, and output format.

Do not hard-code niche assumptions into the platform pipeline.
Do not build a new platform for each niche.
Build one factory. Build many assembly lines.

---

## current implementation notes

The current sports implementation uses one model call per run.
FastAPI receives grounded signals from the retriever and structures the prompt through the 4 cognitive phases.
The platform handles synthesize as deterministic code in `SportsComposer`.

Current pipeline: retrieve → analyze → evaluate → quality_check → compose

- retrieve: external http calls, grounded signal collection
- analyze: one gpt-4o-mini call emitting structured phase output
- evaluate: deterministic confidence calibration based on evidence richness
- quality_check: deterministic internal artifact quality warnings
- compose: synthesize — integrates phase material, assembles `AgentRunExecutionResult`, maps delivery fields

Artifact Quality v1 adds platform-owned `MissingSignals` and `ArtifactQualityWarnings` in `OutputJson`.
These are deterministic internal quality-loop fields.
They are not user-facing.
They do not implement the full future `ArtifactSources` taxonomy.
The full `known_facts`, `ai_interpretations`, `missing_information`, and `excluded_inputs` taxonomy remains deferred.

`SportsQualityChecker` rule 4 normalizes model-emitted signal vocabulary to platform canonical signal names before checking grounded-set membership. This prevents false warnings when the model uses its own vocabulary (e.g. `rest_fatigue`) while the platform retriever grounds the canonical name (`rest_schedule`). Normalization happens in the quality checker — the FastAPI prompt vocabulary is unchanged.

Run Artifact Inspection v1 provides a tenant/user-scoped read-only inspection endpoint for internal artifact quality and cognitive phase review.
It is for platform learning and debugging, not the main user-facing sports read.
Dev Artifact Review Page v1 exposes that endpoint in the sports app at `/dev/artifacts` as a hidden builder review surface.
It is for builder learning, debugging, and quality review, not the main user-facing sports read.

The conceptual artifact template, full phase attribution in separate SQL columns, and vector memory layers are target doctrine — not yet implemented.

See `sports-cognitive-worker-model.md` for the sports-specific pipeline detail.

---

## cognitive skill pack architecture v1

The phase ownership defined above is being layered into a versioned skill pack model.
Two tracks share the same shape but have different runtime status:

- **Claude Code skill packs** — development-time helpers under `dai/.claude/skills/`. Zero runtime dependency.
- **DAI runtime worker packs** — production worker definitions that encode how each phase performs inside an agent run.

The platform must not depend on Claude Code skills at runtime. The first concrete pack is `dai-signal-follow-up-diagnostics`, which lives in the **discern** phase.

See:
- `cognitive-factory/cognitive-skill-pack-architecture-v1.md`
- `cognitive-factory/phases/perceive.md`
- `cognitive-factory/phases/interrogate.md`
- `cognitive-factory/phases/discern.md`
- `cognitive-factory/phases/decide.md`
- `cognitive-factory/phases/synthesize.md`
