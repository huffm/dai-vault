---
title: "Capability Recommendation and Tool Selection Implementation Plan v1"
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
  - platform-architecture
  - tool-gateway
  - planning
related:
  - "02 Platform/architecture/capability-recommendation-and-tool-selection-standard-v1.md"
  - "02 Platform/system-development/work-items/WI-0031-model-assisted-capability-recommendation-and-tool-selection-v1.md"
  - "02 Platform/architecture/tool-gateway-and-agent-permissions-doctrine-v1.md"
---

# capability recommendation and tool selection implementation plan v1

## purpose

Sequence the WI-0031 implementation into six independently reviewable slices. The normative
`capability-recommendation-and-tool-selection-standard-v1` defines the "what"; WI-0031 governs
execution and status; this plan defines the "in what order" and assigns selection-trace field
ownership to slices. No slice is implemented by this planning document.

## context

Planning slice 2026-07-18 established the standard and this decomposition. RC readiness is
integrated (vault main `ffa9e0e`); posture is documentation-only. The model is a semantic
recommender; the Tool Gateway remains the execution authority.

## scope

Included: slice objectives, boundaries, dependencies, verification shape, trace-field ownership,
and promotion criteria. Excluded: any runtime code, model call, registry/schema, telemetry
persistence, CLI/API, scheduler, dashboard, weight tuning, or pilot integration.

## decomposition assessment (result)

One parent work item (WI-0031) with six vertical slices. Behavioral cohesion is high; the model
boundary, policy authority (gates), persistence, and pilot integration are separable review and
rollback units; local doctrine permits one work item across several slices. Promote a slice to a
child work item only when it carries an independent policy authority, persistence, model-spend,
tenant-security, or rollback boundary.

## slice sequence

### Slice 1 -- domain contracts and offline selection trace
- Objective: typed ingredients; capability ontology contract; candidate and accessibility
  states; hard-gate result model; contextual score contract; recipe model; selection trace;
  deterministic fixtures.
- Boundary: no model call, no registry persistence, no network, no runtime integration.
- Trace fields owned: normalized_ingredients, recommended_capabilities (shape),
  resolved_tool_candidates (shape), accessibility_state, hard_gate_results (shape),
  score_components (shape), final_score, deterministic_tie_breakers, selected_tools (shape),
  shadow_recommendations, compiled_recipe (shape), versions (schema/ontology).
- Verification: deterministic fixtures traverse the pure decision path; byte-identical trace
  serialization; hard gate never rescued by score; shadow recommendation retained.
- Dependency: parent WI; repository-backed language/policy-authority decision.

### Slice 2 -- tenant-scoped capability and tool registry resolution
- Objective: resolve capabilities to tenant-approved implementations; expose accessible and
  shadow catalogs; enforce role/station/tenant scopes; retain inaccessible recommendations.
- Boundary: no model call required; no execution; no cross-tenant context unless explicitly
  permitted.
- Trace fields owned: tenant_id, station_id, agent_role, workflow_phase, tool_registry_version,
  resolved_tool_candidates (population), accessibility_state (population).
- Dependency: Slice 1 contracts; the Tool Gateway registry/`ToolDefinition` metadata.

### Slice 3 -- model-assisted ingredient and capability recommender
- Objective: strict model input/output schema; versioned prompt; metering; capability
  recommendation; uncertainty and unmapped-capability output. The model has no execution
  authority.
- Boundary: single model-assisted step; no credentials to the model; no tool authorization.
- Trace fields owned: model_provider, model_name, model_prompt_version, model_schema_version,
  model_call_metering, recommended_capabilities (population), confidence/uncertainty.
- Dependency: Slices 1-2 contracts and catalogs.

### Slice 4 -- deterministic eligibility, ranking, and recipe compiler
- Objective: hard gates; contextual scoring with versioned weight profiles; deterministic
  tie-breaking; bounded recipe compilation; no permission bypass.
- Boundary: gates are hard constraints, never soft weights; no model-generated free-form recipe
  executes without compilation/validation.
- Trace fields owned: selection_policy_version, weight_profile_version, capability_ontology_version,
  hard_gate_results (population), score_components (population), final_score, selected_tools
  (population), compiled_recipe (population).
- Dependency: Slices 1-3; the Tool Gateway policy layer.

### Slice 5 -- execution telemetry and offline evaluation
- Objective: selection-trace persistence; outcome labels; override capture; capability-gap
  measurements; offline weight analysis. No self-modifying production weights.
- Boundary: offline, governed; privacy/redaction doctrine; no live weight mutation.
- Trace fields owned: request_or_run_id, as_of_utc, execution_outcome, operator_override,
  evaluation_labels; the metrics families.
- Dependency: Slices 1-4; existing outcome/evaluation and privacy doctrine.

### Slice 6 -- governed pilot integration
- Objective: integrate one narrow consumer after platform contracts are stable; candidate =
  Daily Evidence Planner tool-applicability planning; preserve niche logic outside platform core;
  compare model recommendations, policy resolution, and operator outcomes.
- Boundary: no sports logic in platform core; pilot is a candidate, not a commitment.
- Dependency: Slices 1-5 stable.

## trace-field ownership summary

Every selection-trace field in the standard is owned by exactly one slice above; version fields
are set by the slice that first populates them. No field is written in two slices without an
explicit hand-off recorded in the parent WI.

## verification (per slice, defined at that slice)

Each slice defines its verification commands in the parent WI before code; contract changes are
proposals until the implementing slice's review. This plan adds no source tests.

## safety / non-actions

No runtime, model, registry, schema, telemetry, CLI, API, scheduler, dashboard, weight tuning, or
pilot code is created by this plan. No niche logic enters platform core. Learning stays offline.

## recommended next slice

WI-0031 Slice 1 (domain contracts and offline selection trace), starting with the
repository-backed language/policy-authority decision, then deterministic fixtures before code.
