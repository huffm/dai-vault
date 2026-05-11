# cognitive skill pack architecture v1

**date:** 2026-05-11
**status:** v1 — foundation slice. defines the doctrine and the first concrete skill pack. broader runtime worker-pack rollout is deferred.
**owns:** the cognitive factory layer of DAI — phase ownership, dual-track skill pack model (build vs runtime), calibration feedback loop.

---

## core claim

DAI is a decision intelligence factory. Cognitive work happens in a small set of stable phases, regardless of niche.

Each phase is a **bounded responsibility**, not a model call. Each phase will eventually be represented as a **skill pack** — a self-contained, inspectable, versioned definition of how that responsibility is performed.

There are two distinct skill pack tracks:

| track | audience | purpose |
|---|---|---|
| Claude Code skill pack | development time | accelerates the human + assistant building DAI. lives in `dai/.claude/skills/`. zero runtime dependency. |
| DAI runtime worker pack | platform runtime | encodes how a cognitive worker performs its phase inside an agent run. lives in services. zero dependency on Claude Code. |

They share shape and vocabulary on purpose — the rhyme is the point. They are not the same dependency.

---

## why two tracks

**Claude Code skill packs** are markdown bundles that the assistant loads when a relevant task surfaces. They make architectural and diagnostic work repeatable across sessions and across people. They are dev-time scaffolding.

**DAI runtime worker packs** are platform concepts that backend services consume to execute cognitive work for tenants. They are production code and config. They must run with no Claude Code present and no markdown loading at runtime.

The platform must not depend on Claude Code skills at runtime. If the build helpers vanished tomorrow, the factory still runs. This separation keeps the factory shippable, auditable, and portable.

---

## phase model

Five cognitive phases. They map cleanly onto the existing `retrieve → analyze → evaluate → quality_check → compose` spine without renaming any code today.

| phase | responsibility | current code mapping |
|---|---|---|
| perceive | stage grounded context, surface anomalies, frame the decision | `SportsRetriever`, `SportsCollector`, prompt `perceive` block |
| interrogate | apply pressure to the initial read — balance, stress, reframe | prompt `interrogate` block |
| discern | judge evidence quality, separate grounded from weak, listen to external signals | `SportsEvaluator`, `SignalQualityEvaluator`, `SignalFollowUpEvaluator`, prompt `discern` block |
| decide | calibrate confidence, set posture, choose voice | `SportsEvaluator` calibration, prompt `decide` block, posture clamp in FastAPI |
| synthesize | integrate validated material into the consumable artifact — no new claims | `SportsComposer` |

Phase ownership is doctrine. Existing pipeline-step names (`retrieve`, `analyze`, `evaluate`, `quality_check`, `compose`) remain intact in code. The phase vocabulary layers on top to give responsibility a stable name even when implementation shifts.

See `phases/perceive.md`, `phases/interrogate.md`, `phases/discern.md`, `phases/decide.md`, `phases/synthesize.md` for per-phase contracts.

---

## what each phase must not do

Phase boundaries matter more than phase contents. The shortest reliable guard against drift is a clear list of forbidden behavior per phase. Each phase doc states its `must not` list. The platform enforces those guards through deterministic code, schema validation, and prompt boundaries.

A condensed view:

- **perceive** must not decide posture, invent missing data, or overstate signal strength.
- **interrogate** must not add new facts, attack for noise, or decide final stance.
- **discern** must not treat interpretation as fact, admit weak evidence as strong, or ignore missing information.
- **decide** must not let confidence exceed evidence, use hype language, or call uncertain reads certain.
- **synthesize** must not invent evidence, override evidence quality, or hide uncertainty surfaced in prior phases.

These are the same constraints already documented in `cognitive-worker-doctrine.md` and `sports-cognitive-worker-model.md`. v1 keeps them, names the owning phase, and points future calibration at the right phase.

---

## DAI runtime worker pack shape (target)

Today, the platform implements phase work through inline code paths and prompt blocks. The eventual shape is a versioned worker pack per phase per niche. The pack is data + code, not a service.

Each runtime worker pack will define:

```text
WorkerDefinition {
  name,                       // e.g. "sports.perceive.v1"
  purpose,                    // one sentence; what this worker owns
  input_contract,             // typed shape consumed (e.g. SportsRetrievalOutput)
  output_contract,            // typed shape produced (e.g. SignalAvailabilityRecord[])
  allowed_tools,              // platform-owned clients this worker may call
  forbidden_behaviors,        // phase-specific `must not` list, machine-checkable when possible
  validators,                 // deterministic schema + invariant checks
  prompt_template,            // if this worker calls a model; null otherwise
  calibration_hooks,          // how this worker's output is measured against outcomes
  example_artifacts           // golden inputs and outputs for regression review
}
```

Supporting types:

```text
WorkerInput        - typed payload consumed by a worker.
WorkerOutput       - typed payload produced by a worker; the schema is enforced before storage.
WorkerRules        - declarative constraints (forbidden vocab, allowed enums, range clamps).
WorkerValidator    - deterministic check function; returns pass + reasons or fail + reasons.
CalibrationHook    - reference to the metric or outcome record that grades this worker over time.
```

**likely future home:** `services/agent-service/cognitive_workers/<phase>/`, mirroring the existing FastAPI service layout. **note:** the source of truth for cognitive work today is .NET (`platform/dotnet/DevCore.Api/AgentRuns/`), not FastAPI. The runtime pack should live wherever the owning code already lives — perceive and discern map mostly to .NET today; only `analyze` (the model call) lives in FastAPI. v1 does not move code. v1 only names ownership.

This is a target shape. Do not generate empty pack scaffolding ahead of need. The first pack should appear when there is real reuse — for example when the second niche (crypto, stocks, kalshi) needs the same phase responsibility expressed differently.

---

## Claude Code skill pack shape

Each Claude Code skill lives under `dai/.claude/skills/<skill-name>/` and uses progressive disclosure:

```text
SKILL.md          - top-level entry. when to use, what to read, output format, guardrails, links.
purpose.md        - the why and the scope of the skill.
inputs.md         - what files, fields, and metadata the skill reads.
outputs.md        - the structured shape the skill must emit.
rules.md          - hard rules the skill must follow.
anti-patterns.md  - common ways the skill can go wrong and how to avoid them.
examples.md       - one or two worked examples.
scripts/          - optional. only added when a deterministic helper is genuinely useful.
```

The skill returns structured markdown for a human reviewer. It does not write code unless asked. It does not invent signals or fields that are not already in the artifact.

---

## first concrete skill pack: dai-signal-follow-up-diagnostics

The first skill pack proves the shape against a real, repeating need: when a sports run completes with partial grounding, a reviewer needs a clean diagnosis of what is missing, why, what to do next, and which phase owns the gap.

The skill reads from the existing artifact inspection surface (`GET /api/agent-runs/{id}/artifact`) and the calibration report markdown files in `dai-vault/04 Products/sports-v1/calibration/`. It writes a structured diagnosis block. It does not change retrieval, scoring, or prompts.

It maps directly to the discern phase: judging signal quality and follow-up routing is discern's job. The skill names that explicitly so the next slice can land in the right place.

See `dai/.claude/skills/dai-signal-follow-up-diagnostics/SKILL.md` for the full pack.

---

## calibration feedback loop

A skill pack — Claude Code or runtime — earns its place only if its output measurably improves the artifact.

For DAI runtime worker packs, calibration flows like this:

1. Each agent run produces a worker output stored in `OutputJson`.
2. Outcome records (`AgentRunOutcome`, `AgentRunEvaluation`) accumulate over time.
3. A periodic calibration pass joins worker outputs to outcomes, by phase.
4. The pass produces a grade per worker pack: did its output predict, support, or mislead the final decision?
5. Grades feed back into the owning pack as a new prompt revision, new validator threshold, or new follow-up rule.

For Claude Code skill packs, calibration is lighter:

1. The skill output is reviewed by the human + assistant pair on real runs.
2. When the skill emits a diagnosis that is wrong, missed, or off-phase, the pack is updated.
3. The pack's `rules.md` and `anti-patterns.md` are the durable home for that learning.

Calibration always updates the responsible pack. It never silently changes the platform pipeline. Pipeline changes happen as a deliberate slice, with the calibration finding as motivation.

---

## flywheel

The factory compounds when:

- phase ownership is stable
- each phase has a single, versioned definition of how it performs
- outcomes grade each phase independently
- gaps name the owning phase so a slice lands in the right place
- niches reuse phase definitions and override only what is genuinely niche-specific

That is the decision intelligence flywheel. v1 names the phases. v1 builds one Claude Code pack. Later slices land the next packs only when reuse is real.

---

## scope guard

v1 does:

- name the five phases and their boundaries
- introduce the Claude Code skill pack vs DAI runtime worker pack split
- document the target runtime worker pack shape
- ship one concrete Claude Code skill (`dai-signal-follow-up-diagnostics`)
- cross-link the new structure into existing flow and cognitive-worker docs

v1 does not:

- rename `retrieve / analyze / evaluate / quality_check / compose` in code
- move any production code into a new folder
- introduce a new runtime abstraction or worker registry
- create skill packs for phases other than the diagnostics use case
- generalize to niches other than sports

The pattern proves itself in sports first. Niche generalization happens after the first runtime worker pack lands.

---

## references

- `cognitive-worker-doctrine.md` — platform doctrine for cognitive work; phase boundaries live there too.
- `sports-cognitive-worker-model.md` — sports pipeline detail and current implementation status.
- `decision-intelligence-model.md` — strategic direction and target artifact shape.
- `orchestration.md` — current four-step pipeline and confidence ownership.
- `current-sports-analysis-flow.md` — end-to-end request path and storage.
- `current-agent-run-contract.md` — db entities, API contracts, signal availability fields.
- `signal-fallback-ladder.md` — first concrete extension of v1: ladder vocabulary, invariants, and concrete classifications.
- `dai/.claude/skills/dai-signal-follow-up-diagnostics/SKILL.md` — the first concrete skill pack.
