# capability recommendation and tool selection standard v1

**status:** active doctrine (normative standard) -- planning-stage; governs WI-0031 implementation slices; no runtime implemented
**date:** 2026-07-18

## purpose

State, in one canonical place, how DAI turns unstructured signals into a governed,
deterministic tool-selection decision: how a model call recommends *capabilities*, how the
deterministic platform resolves those to *tenant-approved tool implementations*, how hard
policy gates and versioned contextual weights select among eligible tools, how a validated
executable recipe is compiled, and how full selection provenance and measured outcomes feed
later offline calibration. The model is a semantic recommender; it is never the execution
authority. This standard is normative; WI-0031 governs its implementation; the implementation
plan sequences the slices; source and tests remain the behavioral truth once code exists.

## problem it solves

DAI already grounds analysis in retrieved signals and governs tool access through a
fail-closed gateway ([[tool-gateway-and-agent-permissions-doctrine-v1]]), and it already has a
selection-provenance dialect for prompts (ingredients -> candidates -> eligibility -> selected
-> policy version -> execution -> outcome -> calibration; ADR 0007, `SignalAvailability`,
`AgentRunOutcome`/`AgentRunEvaluation`). What is missing is one normative contract for the
*general* case: interpreting signals with a model, recommending capabilities, distinguishing
what is executable from what is merely relevant-but-inaccessible, resolving capabilities to
concrete tool implementations under tenant/role/station policy, ranking eligible tools with
governed weights, compiling a bounded recipe, and retaining both what the model recommended
and what the platform permitted. Without this, each niche would re-derive (and quietly widen)
capability boundaries, fork the provenance dialect, or let a model's ask become a grant.

## strategic fit

Platform = factory; agents = workers; tenants = businesses/workspaces; niches = assembly
lines; frontends = packaging; stripe = truth. The factory only scales safely if a worker gets
exactly the capability its station needs and no more, and if every capability gap is measured
rather than silently dropped. This standard is the planning brain in front of the Tool Gateway
capability boundary: the model proposes what *would* help, the deterministic platform decides
what *may* run, and the difference between the two becomes capability-gap telemetry that guides
where to invest. It protects the same long-term stock as the rest of the platform -- buyer and
tenant trust -- by keeping capability explicit, scoped, observable, and never model-granted.

## mental model

Two authorities, one direction of flow:

- The **model** is a bounded semantic recommender. Given normalized signals and a versioned
  capability ontology, it extracts ingredients and recommends capabilities with confidence and
  structured rationale. It never receives credentials, never infers permission from tool
  presence, never emits an executable script, and never authorizes a tool.
- The **deterministic platform** (the Tool Gateway and its policy layer) is the execution
  authority. It owns tenant isolation, agent/station permissions, tool access, side-effect
  authority, data-classification constraints, spend and rate limits, modality/schema
  compatibility, recipe validity, and final execution eligibility.

Capability flows one way: the platform grants a scoped, observable capability to a station; the
station (and the model filling it) cannot grant itself more. The standard preserves BOTH what
the model recommended AND what the platform permitted and selected. A relevant but inaccessible
recommendation is never silently discarded -- it is retained as a shadow recommendation and
becomes capability-gap evidence.

## what it is

- The normative contract for the model-assisted capability recommendation and deterministic
  tool-selection feedback loop.
- The definition of the stages, the typed terminology, the hard-gate set, the contextual
  weight model, the selection trace, the reason/provenance contract, the metrics, and the
  learning boundary.
- An anchor that reuses the Tool Gateway doctrine and the prompt-selection provenance dialect
  rather than forking them.

## what it is not

- Not an implementation. No model call, registry, resolver, recipe compiler, telemetry
  pipeline, schema, service, CLI, API, scheduler, or dashboard is built by this standard.
- Not a claim that any of this is enforced today; it is doctrine over a planned capability.
- Not a general plugin/agent-autonomy layer. The gateway remains a governed boundary.
- Not sports-specific. No niche rule, threshold, tool, or workflow is part of the platform
  standard. The Daily Evidence Planner is recorded only as a possible future pilot consumer.
- Not authorization to enable online self-modifying weights; v1 learning is offline and
  governed.

## platform versus niche boundary

The **platform** owns: the capability ontology; the tool-implementation registry contract; the
recommendation contract; deterministic eligibility and policy resolution; the selection-trace
schema; weight-profile versioning; the recipe-compilation boundary; outcome/feedback telemetry;
tenant and agent permission enforcement; observability and auditability.

**Niche or tenant configuration** owns: available data sources; niche-specific signals and
ingredient vocabulary; permitted capability mappings; station-specific scoring profiles; tool
implementations enabled for that tenant; niche thresholds and output contracts; outcome labels
meaningful to that product.

Niche assumptions never enter platform core. A niche varies by data sources, prompts,
thresholds, scoring rules, and output format -- not by forking the standard.

## relationship to existing prompt selection (reuse, do not fork)

DAI already selects prompts through a provenance chain
(`decision-intelligence-model.md`; ADR 0007 prompt-route attribution;
`AgentRunOutcome` + `AgentRunEvaluation`; `SignalAvailability` per-signal source/quality;
evidence-readiness Gate 3/4 in [[evidence-readiness-gates-v1]]). Tool selection mirrors this
dialect one-to-one and adds the controls tool execution requires:

| Prompt-selection concept | Tool-selection analogue | Additional control tool selection needs |
|---|---|---|
| grounded signals / `SignalAvailability` | normalized ingredients | typed ingredient vocabulary, per-ingredient provenance |
| prompt candidate | recommended capability | capability != concrete tool; capability is implementation-independent |
| candidate eligibility | hard policy gates | permission, tenant isolation, side-effect authority, spend, rate limit, availability |
| selected prompt route | selected tool implementation | resolution to a tenant-approved `ToolDefinition`; recipe compilation |
| `promptRouteProvenance` / policy version | selection trace / policy+weight+ontology+registry versions | tool/capability version, metering, accessibility state per candidate |
| `AgentRunOutcome`/`AgentRunEvaluation` | execution outcome + evaluation labels | operator override, capability-gap disposition, cost/latency-adjusted utility |
| pooled calibration (offline) | offline weight calibration | governed weight profiles; no online self-modification |

Do not create a second provenance dialect where an existing DAI concept can be reused safely.
Where terminology already exists, record the alias and any migration risk.

## terminology (stable vocabulary)

- **Signal** -- raw structured or unstructured information presented to the selection system.
- **Ingredient** -- a normalized, typed feature extracted from signals and relevant to
  capability selection.
- **Capability** -- an implementation-independent action or information need (e.g. document
  parsing, current public search, internal-record retrieval, market-depth lookup, code
  inspection, message delivery). The model primarily recommends capabilities, not vendor tools.
- **Tool implementation** -- a concrete executable registered to satisfy one or more
  capabilities (a `ToolDefinition` reachable only through the Tool Gateway).
- **Accessible catalog** -- capabilities/implementations currently executable under the tenant,
  station, role, policy, spend, and operational context.
- **Shadow catalog** -- known capabilities/implementations that may be semantically relevant
  but are not currently executable.
- **Unmapped capability** -- a recommended capability with no registered platform
  implementation.
- **Eligibility gate** -- a deterministic hard constraint that must pass before a tool may be
  selected.
- **Selection score** -- a versioned contextual score used only among tools that passed every
  hard gate.
- **Recipe** -- an ordered, policy-valid sequence of selected tool invocations and their
  declared dependencies.
- **Shadow recommendation** -- a recommendation retained for measurement even though it cannot
  enter the executable recipe.
- **Selection trace** -- the complete causal record of ingredients, candidates, gates, scores,
  selected tools, blocked tools, model provenance, policy versions, recipe, execution, and
  outcome.
- **Outcome feedback** -- measured evidence used to evaluate whether the recommendation and
  selection were useful.

Refine terms only to align with existing canonical DAI vocabulary; record aliases and migration
risks where terminology already exists.

## normative architecture -- stages

Control flow:

```text
unstructured signals
-> structured ingredient extraction
-> semantic capability recommendations
-> capability-access and policy resolution
-> deterministic ranking among eligible tool implementations
-> executable recipe compilation
-> tool execution
-> outcome evaluation
-> selection telemetry
-> offline weight calibration and later learning
```

### stage A -- signal normalization (deterministic)

Inputs may include unstructured text, structured request context, tenant identity, agent role,
station/workflow phase, data modality, declared objective, prior execution context, and
policy/registry versions. Raw signals must not directly grant tool authority. Normalization is
deterministic and version-stamped.

### stage B -- model-assisted ingredient and capability recommendation

The model may extract normalized ingredients; recommend capabilities; provide confidence;
provide stable reason codes or structured rationale inputs; identify uncertainty; and identify
potentially missing capabilities. It receives a bounded, versioned capability ontology or
candidate catalog. It must not receive credentials or infer permission from tool presence. Its
output uses a strict structured schema. It does not produce an executable script and does not
authorize a tool.

### stage C -- capability and implementation resolution (deterministic)

The resolver maps each recommended capability to tenant-approved implementations, inaccessible
implementations, unavailable implementations, or unknown/unmapped capabilities, and records all
resolution results. Resolution is tenant-scoped; cross-tenant context is not used unless
explicitly permitted.

### stage D -- hard policy gates (deterministic, never soft weights)

At minimum, evaluate: tenant permission; agent-role permission; station/workflow-phase
permission; side-effect authority; cross-tenant isolation; data-classification compatibility;
modality/schema compatibility; credential/configuration readiness; spend authorization; rate
limits/quotas; geographic/regulatory restrictions when applicable; tool health/operational
availability; recipe dependency compatibility. A candidate failing any required gate is
ineligible; no score may rescue it. Gates reuse and extend the gateway's `AllowedProtocolNodes`
plus `ToolDefinition` metadata (kind, transport, cost class, idempotency, secrets scope).

### stage E -- weighted ranking among eligible implementations

Only eligible implementations may be ranked. The model distinguishes tool base priors,
contextual selection features, versioned weight profiles, a final score, and deterministic
tie-breakers. Candidate score components include semantic relevance, ingredient coverage,
historical utility, observed reliability, evidence quality, operational freshness, latency,
cost, duplication/redundancy, and downstream compatibility. No production weight values are
mandated in this planning stage; initial weights are manually governed, versioned as a
`weight_profile_version`, and each score contribution is recorded in the trace. There is no one
permanent global "tool weight"; utility is contextual:

```text
tool utility =
  tool implementation
  x ingredients
  x tenant
  x station
  x agent role
  x workflow phase
  x policy version
  x weight-profile version
```

### stage F -- recipe compilation (deterministic)

The compiler converts selected eligible tools into a bounded executable recipe and validates
step ordering; required inputs; declared outputs; dependencies; authority at each step; cost and
call ceilings; retry rules; side-effect boundaries; stopping conditions; fallback rules; and
idempotency expectations where applicable. No model-generated free-form recipe is executable
without deterministic compilation and validation.

### stage G -- execution and outcome capture (future runtime; contract defined here)

Execution is out of scope for WI-0031, but the standard defines the future trace: selected
implementation; tool/capability version; inputs and ingredient references; start/completion
time; status; latency; metered cost; call counts; retries; result availability; policy
exceptions; operator intervention; downstream consumption.

### stage H -- evaluation and feedback (offline, governed)

Record whether the recommendation was relevant; the selected tool succeeded; the result was
useful; the result was consumed downstream; an operator overrode the choice; another eligible
tool would have been preferable; an inaccessible recommendation represented a genuine capability
gap; policy blocked a useful tool; or cost/latency outweighed utility. The feedback loop is
initially offline and governed. **v1 must not self-modify production weights from live
outcomes.**

## component relationships and data-flow boundary

(No separate architecture note is created; component relationships and data flow live here to
avoid duplicated content -- see WI-0031 decomposition rationale.)

- Functional core (deterministic, pure): ingredient/capability/candidate/accessibility models;
  resolution; hard gates; contextual scoring; recipe model; selection trace; reason renderers.
- Model boundary (stage B): the only model-assisted step; strict input/output schema; metered;
  no credentials; no authority.
- Deterministic authority: the Tool Gateway and its policy layer own gates F/execution
  eligibility; the recommender never bypasses it.
- Telemetry/persistence: the selection trace and outcome labels are persisted by a later slice;
  privacy/redaction doctrine applies.
- Pilot consumers (e.g. Daily Evidence Planner) integrate only after platform contracts are
  stable and keep niche logic outside platform core.

## capability disposition contract (accessible / inaccessible / unmet)

Every semantically recommended capability receives exactly one disposition. At minimum:
`SELECTED`; `ELIGIBLE_NOT_SELECTED`; `INACCESSIBLE_PERMISSION`;
`INACCESSIBLE_SIDE_EFFECT_AUTHORITY`; `INACCESSIBLE_COST_AUTHORITY`; `INACCESSIBLE_RATE_LIMIT`;
`UNAVAILABLE_CONFIGURATION`; `UNAVAILABLE_OPERATIONAL`; `INCOMPATIBLE_MODALITY`;
`INCOMPATIBLE_SCHEMA`; `UNMAPPED_CAPABILITY`; `DUPLICATE_OR_REDUNDANT`; `RECIPE_CONFLICT`;
`MODEL_RECOMMENDATION_REJECTED_BY_POLICY`. Names may be refined, but the distinctions must
remain measurable.

A blocked recommendation retains: capability ID; proposed tool implementation when resolvable;
semantic relevance; matched ingredients; model and prompt provenance; availability state; gate
failure; policy version; tenant and station context; and whether an operator later enabled or
substituted another capability. This is the capability-gap telemetry. Inaccessible tools are
never exposed to execution merely to collect feedback.

## selection-trace contract (versioned)

At minimum (final field ownership assigned per slice in the plan):

```text
selection_id; tenant_id; station_id; agent_role; workflow_phase; request_or_run_id; as_of_utc;
selection_policy_version; weight_profile_version; capability_ontology_version;
tool_registry_version; model_provider; model_name; model_prompt_version; model_schema_version;
model_call_metering; normalized_ingredients; recommended_capabilities; resolved_tool_candidates;
accessibility_state; hard_gate_results; score_components; final_score;
deterministic_tie_breakers; selected_tools; shadow_recommendations; compiled_recipe;
operator_override; execution_outcome; evaluation_labels
```

Sensitive values follow existing privacy/logging doctrine. The trace stores no secrets,
credentials, raw authorization tokens, or unrestricted tenant data. The standard identifies
required vs optional fields, cardinality, versioning, retention considerations, privacy
concerns, and the future component owning each field (assigned in the implementation plan).

## deterministic reason and provenance contract

Canonical decisions use stable reason codes, structured parameters, stable ordering, and
versioned deterministic renderers. Unconstrained prose never becomes executable policy;
human-readable explanations are rendered from the canonical trace. For each selected tool the
system can answer: which ingredients activated this capability; which hard gates passed; which
alternatives were considered; which candidates were inaccessible; which weights contributed to
the score; why this tool outranked another eligible tool; which policy/registry versions were
used; what happened after execution. For each inaccessible recommendation it can answer: why the
capability was relevant; why it could not execute; whether the limitation was permission,
configuration, cost, availability, or platform coverage; and how often the gap has occurred.

## metrics and feedback-loop standard

Definitions only (no dashboard, no storage). Recommendation quality: recommendation precision;
acceptance rate; irrelevant-recommendation rate; unmapped-capability rate. Availability/policy:
executable-recommendation rate; shadow-recommendation rate; blocked-recommendation rate by
reason; capability-gap recurrence; policy-friction rate; configuration-gap rate. Selection
behavior: selection rate by capability and tool; tool utility by ingredient combination;
operator override rate; deterministic fallback rate; tie-break frequency; tool redundancy rate.
Execution quality: success rate; useful-result rate; downstream-consumption rate; latency;
metered cost; cost-adjusted utility; latency-adjusted utility; retry/failure rates. Learning
readiness: labeled selection count; label coverage; model-vs-operator disagreement; outcome
completeness; selection regret where measurable; distribution drift by tenant/station/niche/
policy version. Each metric documents its numerator, denominator, and required trace fields.

## learning and fine-tuning boundary

Fine-tuning is explicitly deferred until sufficient high-quality telemetry exists. Progression:

```text
versioned prompt and schema
-> deterministic policy gates
-> manually governed weight profiles
-> complete selection traces
-> operator and outcome labels
-> offline evaluation
-> offline weight recalibration
-> candidate learned reranker or classifier
-> fine-tuning only when justified
```

Future learning options are evaluated without premature selection: calibrated rules; logistic
or tree-based classifier; reranker; contextual bandit; small applicability model; LLM fine-tune.
The eventual learned target may resemble `P(tool useful | ingredients, tenant, station, role,
phase, policy)`. No learning component may grant permission, bypass policy, or self-expand
capability access.

## failure and fallback policy

Define future behavior for: malformed model output; model timeout; model refusal; unknown
capability recommendation; stale registry; policy-version mismatch; no eligible tools; multiple
equivalent eligible tools; recipe-compilation failure; selected-tool runtime failure; telemetry
write failure. Default principles: fail closed on permission, tenant, data-classification,
spend, and side-effect uncertainty; never interpret model confidence as authority; never
silently drop blocked recommendations; never fabricate an executable tool; never silently switch
to an unapproved implementation; allow a deterministic fallback only when that fallback is
explicitly defined, versioned, and authorized for the station; record whether the outcome came
from model-assisted selection or deterministic fallback.

## security, tenancy, and safety

Require: tenant-scoped catalogs and tool resolution; no cross-tenant recommendation context
unless explicitly permitted; no model visibility into credentials; no implicit capability
inheritance; explicit role and station scopes; least privilege; call ceilings; spend ceilings;
rate limits; auditability; trace redaction; and policy-version provenance. Documented threat
cases: prompt injection requesting an unauthorized tool; recommendation of an inaccessible
high-impact capability; tenant-context leakage; model hallucination of a nonexistent tool;
weight manipulation; telemetry poisoning; operator-override abuse; and feedback loops
reinforcing low-quality selections. Each threat maps to a fail-closed control above.

## truth hierarchy

1. Observed runtime behavior and tests (once implemented -- none yet).
2. Source code (the gateway/policy today; the future planner core).
3. Explicit contracts and vault docs (this standard; the Tool Gateway doctrine; ADR 0007;
   the security charter; evidence-readiness gates).
4. Work item WI-0031 and slice handoffs.
5. Generated graphs and prior assumptions -- navigation only.

A capability or permission claim is only true if the policy and its tests say so; the model's
recommendation is never authority.

## approved uses

- Designing or reviewing a model-assisted tool-selection capability against one contract.
- Adding a capability/tool: declare the capability, register a tenant-scoped `ToolDefinition`
  with explicit `AllowedProtocolNodes`, and rely on fail-closed gates; the model only recommends.
- Measuring capability gaps from shadow recommendations to guide platform investment.
- Sequencing implementation via the WI-0031 slices.

## disallowed uses

- Letting a model's request expand permissions, or treating confidence as authority.
- Executing an inaccessible tool to collect feedback, or fabricating a tool.
- Representing a hard gate as a soft weight, or a proposed cohort/recipe as an authorization.
- Embedding any niche (e.g. sports) rule, threshold, tool, or workflow in platform core.
- Enabling online self-modifying weights in v1.

## acceptance criteria

- Model recommendation and deterministic execution authority are separated and cannot be
  confused; the model never authorizes a tool.
- Capability and concrete tool implementation are distinct concepts.
- Accessible, inaccessible, unavailable, and unmapped recommendations are all retained and
  measurable; nothing relevant is silently dropped.
- Hard gates and contextual weights are distinct; no score rescues an ineligible tool.
- The contextual scoring and versioning model (policy/weight/ontology/registry versions) is
  documented, with per-contribution recording.
- The selection trace can explain every recommendation and selection, and every inaccessible
  recommendation.
- Capability-gap telemetry and outcome/feedback metrics are defined.
- Fine-tuning is explicitly deferred; tenant and agent permissions remain deterministic.
- No runtime behavior changed and no niche logic entered platform core.

## risks and failure modes

- Provenance-dialect fork: a second trace vocabulary -> mitigated by the reuse table above.
- Implicit power creep via model recommendation -> blocked by gates being declared policy.
- Capability-gap loss: dropping inaccessible recommendations -> blocked by shadow retention.
- Weight opacity or a single global weight -> mitigated by contextual, versioned, recorded
  contributions.
- Niche leakage into platform core -> blocked by the platform/niche boundary.
- Over-claiming enforcement (citing deferred gateway controls as live) -> mitigated by the
  current-vs-deferred framing inherited from the Tool Gateway doctrine.

## slice 4 delivered semantics (2026-07-20, deterministic ranking + bounded plan)

The deterministic decision layer between resolved recommendations and any future execution
is implemented as a pure domain component (`DeterministicPlanBuilder`; see the closeout
[[capability-selection-deterministic-plan-building-slice-4-2026-07-20-v1]]). Normative
semantics now fixed:

- **hard rule vs score:** a hard rule is yes/no; a failure is never rescued by a score. A
  shadow candidate is `blocked`; an accessible candidate with any unevaluated required
  dimension is `authorization_pending` (fail closed, `NotEvaluated != Allowed`) and is
  never scored; only fully-evaluated accessible candidates are scored.
- **weight profile:** a named, versioned profile weights a closed set of known
  score-component names over eligible candidates only; duplicate, unknown, missing-required,
  and non-finite weights are rejected; absent facts contribute the unfavorable floor and
  unknown components are ignored (never favorable); the profile name/version is in the
  trace. One fixture-backed profile (`deterministic-plan-ranking/1.0`) is delivered.
- **bounded plan:** ordered by score desc then stable identity (capability, tool); max
  steps >= 0 (0 = honest empty plan; negative rejected); no duplicate tool step; blocked
  and inaccessible candidates stay in the trace but outside the plan.
- **no authority:** plan status is `planned_not_authorized` / `authorization_pending` /
  `blocked` (never approved/authorized/executable/ready_to_run); steps carry no
  credential/payload/argument/callback/endpoint; the authority ledger is all-false; the
  **Tool Gateway remains the runtime permission authority** and is re-checked at any future
  execution.

## deferred decisions

- Concrete production weight VALUES beyond the delivered fixture profile (Slice 5 tuning).
- Selection-trace persistence store, retention, and redaction implementation (Slice 5).
- Learned reranker/classifier/bandit choice and any fine-tuning (post-telemetry).
- The first pilot consumer and its niche adapter (Slice 6; Daily Evidence Planner is a
  candidate, not a commitment).
- Per-station (vs stage-level) gateway enforcement and cost-class rate limiting inherit the
  Tool Gateway doctrine's deferred list.

## related docs

- [[tool-gateway-and-agent-permissions-doctrine-v1]] -- the deterministic authority this
  standard builds on.
- `02 Platform/architecture/security-and-permissions.md` -- the permission charter.
- `02 Platform/architecture/orchestration.md` -- tools are shared platform infrastructure.
- `02 Platform/architecture/decision-intelligence-model.md` -- the provenance/outcome dialect.
- `02 Platform/architecture/governance/evidence-readiness-gates-v1.md` -- Gate 3/4 learning
  discipline.
- `02 Platform/decisions/0007-prompt-route-attribution-contract.md` -- the prompt-selection
  provenance analogue.
- `02 Platform/system-development/work-items/WI-0031-model-assisted-capability-recommendation-and-tool-selection-v1.md` -- the governing work item.
- `06 Execution/plans/capability-recommendation-and-tool-selection-implementation-plan-v1.md` -- the slice map.

## recommended next slice

WI-0031 Slice 1 (domain contracts and offline selection trace): define typed ingredients, the
capability ontology contract, candidate/accessibility states, the hard-gate result model, the
contextual score contract, the recipe model, the selection trace, and deterministic fixtures --
offline only, no model call, no registry persistence, no network, no runtime integration.
