---
title: "WI-0031 Model-Assisted Capability Recommendation and Tool Selection v1"
type: "plan"
date: "2026-07-18"
status: "in-progress"
project: "DAI"
slice: "WI-0031 Model-Assisted Capability Recommendation and Tool Selection v1"
repos:
  dai: "unchanged"
  dai-vault: "docs-only"
tags:
  - system-development
  - work-item
  - platform-architecture
  - tool-gateway
  - observability
related:
  - "02 Platform/architecture/capability-recommendation-and-tool-selection-standard-v1.md"
  - "02 Platform/architecture/tool-gateway-and-agent-permissions-doctrine-v1.md"
  - "06 Execution/plans/capability-recommendation-and-tool-selection-implementation-plan-v1.md"
  - "02 Platform/decisions/0007-prompt-route-attribution-contract.md"
---

# WI-0031 Model-Assisted Capability Recommendation and Tool Selection v1

Category: platform architecture + orchestration contract + observability planning. This work
item is the governing parent for a model-assisted capability recommendation and deterministic
tool-selection feedback loop. This slice completes planning and documentation only; no runtime
component is implemented. Status stays `in-progress`.

## problem

DAI grounds analysis in retrieved signals and governs tool access through a fail-closed gateway
(`02 Platform/architecture/tool-gateway-and-agent-permissions-doctrine-v1.md`), and it has a
prompt-selection provenance dialect (ADR 0007; `SignalAvailability`; `AgentRunOutcome`/
`AgentRunEvaluation`). There is no single normative contract for the general case: interpreting
unstructured signals with a model, recommending capabilities, distinguishing executable from
inaccessible recommendations, resolving capabilities to tenant-approved tool implementations,
applying deterministic gates, ranking eligible tools with versioned weights, compiling a bounded
recipe, retaining full provenance, evaluating outcomes, and feeding offline calibration. Absent
this, niches would re-derive and quietly widen capability boundaries, fork the provenance
dialect, or let a model's ask become a grant.

## desired behavior

A durable platform standard and governed decomposition exist so future slices can implement the
loop deterministically: the model recommends capabilities (never authorizes tools); the Tool
Gateway remains the execution authority; every recommendation receives a measurable disposition;
inaccessible-but-relevant recommendations are retained as capability-gap telemetry; hard gates
and contextual weights are distinct; a versioned selection trace explains every decision;
learning is offline and governed; no niche logic enters platform core.

## affected surfaces

- New normative standard: `02 Platform/architecture/capability-recommendation-and-tool-selection-standard-v1.md`.
- New implementation plan: `06 Execution/plans/capability-recommendation-and-tool-selection-implementation-plan-v1.md`.
- This work item and the system-development MOC entry.
- Planning closeout report and the rolling `06 Execution/handoffs/current-slice.md`.
- No DAI runtime/source/prompt/registry/schema surface is touched in this slice.

## non-goals

No implementation of the model call, capability ontology/registry, tool resolver, hard-gate
engine, ranking, recipe compiler, telemetry persistence, database schema, runtime service, CLI,
API, scheduler, dashboard, weight tuning, or Daily Evidence Planner integration. No change to
decision logic, prompt logic, reconciliation, calibration, confidence doctrine, buyer-facing
semantics, or agent permissions. No sports-specific rule, threshold, tool, or workflow in the
platform standard. No online self-modifying weights.

## authority model (normative decision)

Two authorities, one direction of flow: the model is a bounded semantic recommender (no
credentials, no permission inference, no executable script, no tool authorization); the
deterministic platform (Tool Gateway + policy) owns tenant isolation, permissions, tool access,
side-effect authority, data-classification, spend/rate limits, modality/schema compatibility,
recipe validity, and final execution eligibility. The standard preserves both what the model
recommended and what the platform permitted/selected; relevant-but-inaccessible recommendations
are never silently dropped.

## normative architecture (summary; full text in the standard)

Stages A signal normalization -> B model-assisted ingredient/capability recommendation ->
C capability/implementation resolution -> D hard policy gates (never soft weights) -> E weighted
ranking among eligible implementations (contextual, versioned) -> F recipe compilation ->
G execution + outcome capture (future) -> H offline evaluation/feedback (no online
self-modification). Reuses the Tool Gateway deterministic authority and the prompt-selection
provenance dialect; adds permission, tenant, side-effect, cost, availability, and rate-limit
controls.

## inputs and outputs / data and telemetry contracts

Inputs: normalized signals, tenant/role/station/phase context, capability ontology version,
tool registry version, policy and weight-profile versions. Outputs: recommended capabilities
with confidence; resolved candidates with accessibility state; hard-gate results; scored
eligible tools; a compiled recipe (future execution); a versioned selection trace; capability-gap
telemetry; outcome/feedback metrics. The selection-trace and metrics contracts are defined
normatively in the standard; field ownership is assigned per slice in the plan.

## deterministic and model-assisted boundaries

Only stage B is model-assisted, under a strict input/output schema, metered, with no credentials
and no authority. Everything else (normalization, resolution, gates, scoring, recipe
compilation, trace, reason rendering) is deterministic. No model-generated free-form recipe is
executable without deterministic compilation and validation.

## privacy and tenant-isolation requirements

Tenant-scoped catalogs and resolution; no cross-tenant recommendation context unless explicitly
permitted; no model visibility into credentials; no implicit capability inheritance; least
privilege; call/spend ceilings; rate limits; auditability; trace redaction (no secrets, tokens,
or unrestricted tenant data in the trace); policy-version provenance.

## failure behavior

Fail closed on permission, tenant, data-classification, spend, and side-effect uncertainty;
never treat model confidence as authority; never drop blocked recommendations; never fabricate a
tool; never silently switch to an unapproved implementation; deterministic fallback only when
explicitly defined, versioned, and station-authorized; record whether the outcome came from
model-assisted selection or deterministic fallback.

## metrics

Recommendation quality, availability/policy, selection behavior, execution quality, and
learning-readiness metric families are defined in the standard with numerator, denominator, and
required trace fields. No dashboard or storage is implemented in this slice.

## decomposition (assessment result)

One parent governed work item (this WI-0031) with six implementation slices. Rationale: high
behavioral cohesion (one capability-selection contract); each slice is independently testable and
produces a meaningful artifact; policy authority (gates) and the model boundary are separable;
persistence/telemetry and pilot integration deserve later review boundaries; local doctrine
permits one work item across several slices. Promote a slice to a child work item only when it
carries an independent policy authority, persistence, model-spend, tenant-security, or rollback
boundary. Full slice definitions live in the implementation plan.

- Slice 1: domain contracts and offline selection trace (no model call, no persistence, no network)
  -- DELIVERED 2026-07-18. Language/policy-authority decision: **C#/.NET** (the deterministic
  authority this WI reuses is the .NET Tool Gateway `ToolDefinition`/`AllowedProtocolNodes`/cost-class/
  idempotency; no Python capability-selection authority exists). Implemented in the pure `DevCore.Domain`
  project (no i/o, no gateway dependency yet): `platform/dotnet/DevCore.Domain/CapabilitySelection/
  CapabilitySelectionCore.cs` (CapabilityDisposition enum + NormalizedIngredient/HardGateResult/
  ScoreComponent/ToolCandidateInput/ScoredCandidate/SelectionTrace records + the pure BuildTrace core +
  canonical JSON). Tests: `platform/dotnet/DevCore.Api.Tests/CapabilitySelection/CapabilitySelectionCoreTests.cs`
  (7 xUnit tests: highest-eligible-selected; no-score-rescues-a-failed-gate; inaccessible-retained-as-shadow;
  byte-identical serialization + capture/screening not authorized; deterministic order). dai commit
  `5def141` core + `69cee8b` review correction (key selection by (capability,tool); additive;
  DevCore.Domain builds 0 warnings; full DevCore.Api.Tests 1286 passed / 0 failed).
- Slice 2: tenant-scoped capability and tool registry resolution (accessible + shadow catalogs)
  -- DELIVERED 2026-07-18 (dai commit `b31cc69`). Authority decision: the resolver **consumes** the
  canonical policy seams (`IProtocolToolAccessPolicy.IsAllowed`, `IToolRegistry.TryGet`), never
  re-implements them (IsAllowed is non-trivial: station cards + stage sentinels + AllowedProtocolNodes),
  and creates no second permission authority. Pure resolver + contracts in `DevCore.Domain`
  (`CapabilityToolResolver.cs`: CapabilityRecommendationInput/CapabilityRegistration/ToolDefinitionFacts/
  CandidateAccessFacts/ResolutionContext/ResolvedCandidate/ResolutionResult + deterministic `Resolve` +
  `ToToolCandidateInputs` Slice-1 projection). Thin adapter in `DevCore.Api`
  (`CapabilityResolutionGatewayAdapter.cs`) projects the real registry/policy into the resolver's
  supplied facts. Tenant/role/side-effect/cost/rate/modality/schema access are SUPPLIED FACTS (gateway
  v1 enforces only AllowedProtocolNodes; the rest are declarative/deferred per doctrine), so the
  resolver never invents them. 21 tests (19 resolver + 2 adapter): registration one/many, shared tool
  across capabilities, tenant/role/node/side-effect/cost/rate denials -> shadow, config/operational
  unavailable, unmapped, missing-definition, duplicate/blank/non-finite handling, determinism, catalog
  exclusivity, tenant isolation (2 contexts), Slice-1 composition (denied candidate never selectable),
  empty result. Full DevCore.Api.Tests **1307 passed / 0 failed**; additive (no project-file change).
- Slice 3: model-assisted ingredient and capability recommender (strict schema, versioned prompt, metering).
- Slice 4: deterministic eligibility, ranking, and recipe compiler (hard gates, contextual scoring, tie-breaks).
- Slice 5: execution telemetry and offline evaluation (trace persistence, labels, gap metrics; no online self-modification).
- Slice 6: governed pilot integration (one narrow consumer; Daily Evidence Planner is a candidate; niche logic stays outside core).

## acceptance criteria

- The normative standard exists in the canonical vault taxonomy and separates model recommendation
  from deterministic execution authority.
- Capability and tool implementation are distinct; accessible/inaccessible/unavailable/unmapped
  recommendations are all retained and measurable.
- Hard gates and contextual weights are distinct; the versioned scoring model is documented.
- The selection trace can explain every recommendation, selection, and inaccessible recommendation.
- Capability-gap telemetry and outcome/feedback metrics are defined; fine-tuning is deferred.
- Tenant and agent permissions remain deterministic and enforceable; no niche logic in platform core.
- The six implementation slices are independently reviewable; a Daily Evidence Planner pilot is
  recorded without introducing sports logic.
- No runtime behavior changed; all changes are local, vault-only, and documented.

## test plan (contract/fixture strategy; no source tests this slice)

Slice 1 will define deterministic fixtures before code: ordinary recommendation with a mix of
eligible/inaccessible/unmapped capabilities; hard-gate failure not rescued by score; shadow
recommendation retained; deterministic tie-break; recipe-compilation validation; repeated
byte-identical trace serialization; policy/weight-version change visibility. This planning slice
adds no source tests; it records the strategy in the standard and plan.

## implementation notes

Reuse, do not fork: the Tool Gateway (`ToolGateway`, `IProtocolToolAccessPolicy`,
`AllowedProtocolNodes`, `ToolDefinition` metadata) is the deterministic authority; the prompt
provenance dialect (ADR 0007; `SignalAvailability`; `AgentRunOutcome`/`AgentRunEvaluation`;
evidence-readiness Gate 3/4) is the trace/outcome spine. The language for the future core is
deferred to Slice 1's repository-backed authority analysis (do not mandate a language here).
Contract changes remain proposals in the standard/plan until their implementing slice's review.

## docs to update

- `02 Platform/system-development/MOC - DAI System Development.md` (register WI-0031).
- Standard and plan are created new in this slice; no historical doc is rewritten.

## verification commands

- strict planning snapshot: `pwsh scripts/dev/planning/build-next-slice-snapshot.ps1 -Strict -OutputPath <scratch>` (exit 0, 0 warnings)
- OKF/front-matter + related-link validation on the new 06 Execution docs and this WI
- `git diff --check`; staged-allowlist print; protected-file hash open/close comparison
- machine-path / secret / private-URL scans over all new artifacts

## risks

Provenance-dialect fork (mitigated by the reuse table); niche leakage into platform core
(mitigated by the platform/niche boundary); over-scoping planning into implementation (bounded by
non-goals and the write boundary); mis-stating deferred gateway controls as live (mitigated by
current-vs-deferred framing inherited from the Tool Gateway doctrine).

## links

- work item: WI-0031 (ADO: AB#— when wired)
- branch: `wi/0031-model-assisted-capability-recommendation-and-tool-selection` (dai-vault; vault-only, docs-only WI)
- pr: — (merged direct convention; not pushed/merged this slice)
- commits: — (recorded in the closeout at slice close)
- tests: none this slice (contract/fixture strategy defined for Slice 1)
- verification notes: see the planning closeout report and current-slice handoff
- docs updated: standard, implementation plan, MOC, closeout, current-slice (this slice)
- lessons: reuse the existing provenance dialect and Tool Gateway authority rather than forking a
  parallel tool-selection vocabulary; keep the model a recommender and the gateway the authority

## final handoff requirements

Slice handoff appended to `06 Execution/handoffs/current-slice.md`; definition of done in
`implementation-lifecycle` checked for a planning slice; disposition recorded as: standard and
implementation plan complete; runtime implementation not started; local branch and commits only;
not integrated. Status remains `in-progress`.
