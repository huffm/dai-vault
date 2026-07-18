---
title: "Capability Recommendation and Tool Selection Standard Planning Closeout v1 (2026-07-18)"
type: "evidence-report"
date: "2026-07-18"
status: "complete"
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
  - "02 Platform/system-development/work-items/WI-0031-model-assisted-capability-recommendation-and-tool-selection-v1.md"
  - "02 Platform/architecture/capability-recommendation-and-tool-selection-standard-v1.md"
  - "06 Execution/plans/capability-recommendation-and-tool-selection-implementation-plan-v1.md"
---

# capability recommendation and tool selection standard planning closeout v1

## purpose

Record what the 2026-07-18 documentation-first planning slice for WI-0031 actually completed:
a governing parent work item, a normative standard, a six-slice implementation plan, a MOC
registration, and this closeout. No runtime component was implemented.

## context

Executed on the vault-only branch `wi/0031-model-assisted-capability-recommendation-and-tool-selection`
from vault main `ffa9e0e` (RC readiness integrated; DAI main `c6166e2` untouched). Planning and
documentation only, per the finalization authorization.

## scope

Included: WI-0031 intake and decomposition; the normative standard; the implementation plan;
the MOC entry; verification; this closeout; the current-slice append. Excluded: any runtime,
model, registry, resolver, gate engine, ranking, recipe compiler, telemetry, schema, service,
CLI, API, scheduler, dashboard, weight tuning, or Daily Evidence Planner integration.

## key decisions

1. **Two authorities, one flow.** The model is a bounded semantic recommender; the deterministic
   Tool Gateway remains the execution authority. The standard preserves both what the model
   recommended and what the platform permitted/selected; relevant-but-inaccessible
   recommendations are retained as capability-gap telemetry, never silently dropped.
2. **Reuse, do not fork.** The standard mirrors the existing prompt-selection provenance dialect
   (ADR 0007; `SignalAvailability`; `AgentRunOutcome`/`AgentRunEvaluation`; evidence-readiness
   Gate 3/4) and builds on the Tool Gateway doctrine (`ToolGateway`, `IProtocolToolAccessPolicy`,
   `AllowedProtocolNodes`, `ToolDefinition` metadata) rather than inventing a second dialect. A
   reuse table maps each prompt-selection concept to its tool-selection analogue and the extra
   control tool execution requires (permission, tenant, side-effect authority, cost, availability,
   rate limit).
3. **Capability != tool.** The model recommends implementation-independent capabilities; the
   deterministic resolver maps them to tenant-approved tool implementations, with a measurable
   disposition for every recommendation (SELECTED / ELIGIBLE_NOT_SELECTED / INACCESSIBLE_* /
   UNAVAILABLE_* / INCOMPATIBLE_* / UNMAPPED_CAPABILITY / DUPLICATE_OR_REDUNDANT / RECIPE_CONFLICT
   / MODEL_RECOMMENDATION_REJECTED_BY_POLICY).
4. **Hard gates are not weights.** Stage D gates are hard constraints; no selection score rescues
   an ineligible tool. Stage E ranks only eligible tools with versioned contextual weights and no
   single global "tool weight".
5. **Platform vs niche boundary preserved.** No sports rule/threshold/tool/workflow enters
   platform core; the Daily Evidence Planner is recorded only as a candidate future pilot.
6. **Learning stays offline.** Fine-tuning is deferred until sufficient high-quality telemetry
   exists; v1 does not self-modify production weights.
7. **No separate architecture note.** Component relationships and data flow fold into the
   normative standard to avoid the duplicated-content anti-pattern (section 20 of the
   authorization permits recording this decision).

## evidence

- Pre-write gate (branch-before-write): paths resolved and repos verified as siblings; dai
  `c6166e2` (0/0), vault `ffa9e0e` (0/0), readiness tip `9af32bd` contained in vault main;
  working-tree drift classified and disjoint (`.obsidian/graph.json` generated,
  `CLAUDE.md` BOM-only, `Welcome.md` deleted-operator-dispositioned, two documented untracked
  files); strict planning snapshot exit 0 / 19 WIs / 0 warnings / 0 continuations; WI-0031 derived
  as the first free id after registered (0001-0013, 0020-0025) and reserved (0014-0019, 0026-0030);
  branch `wi/0031-model-assisted-capability-recommendation-and-tool-selection` created from
  `ffa9e0e` and verified to originate there; only then were files written.
- Branch topology: vault-only, doctrine-consistent for a docs-only work item per
  `work-item-traceability.md` (per-artifact naming, not dual-repo-mandatory) and the established
  vault-only-branch precedent. The DAI repository is content-identical to `c6166e2` (only the
  documented csproj phantom present); no DAI branch created.
- Protected/classified path baselines captured at open and re-verified at close (graph.json
  `b3d68588`, CLAUDE.md `9127e464`, preflight manifest `68948ebd`, synopsis `25835e6c`, dai
  csproj `63ef2488`).

## safety / non-actions

0 model calls; 0 paid calls; 0 source-readiness/odds/market calls; 0 services/containers started;
0 database reads/writes; 0 application-data writes; 0 runs; 0 reconciliations; 0 settlements; 0
runtime/source/prompt/registry/schema/config/permission changes; 0 skills created; 0 schedules; 0
pushes/merges/PRs. No decision logic, prompt logic, calibration, confidence doctrine, or
buyer-facing semantics changed. No niche logic entered platform core.

## files created/changed (this slice, vault-only, on the WI-0031 branch)

- created `02 Platform/system-development/work-items/WI-0031-model-assisted-capability-recommendation-and-tool-selection-v1.md`
- created `02 Platform/architecture/capability-recommendation-and-tool-selection-standard-v1.md`
- created `06 Execution/plans/capability-recommendation-and-tool-selection-implementation-plan-v1.md`
- created `06 Execution/reports/capability-recommendation-and-tool-selection-standard-planning-closeout-2026-07-18-v1.md` (this file)
- modified `02 Platform/system-development/MOC - DAI System Development.md` (register WI-0031)
- append-only `06 Execution/handoffs/current-slice.md`

## follow-up (recorded, not actioned)

- The system-development MOC registry currently stops at WI-0023; WI-0024 and WI-0025 are
  integrated but not registered in the MOC. This pre-existing gap is out of scope for WI-0031 and
  is recorded here for a future hygiene slice rather than broadening this slice.

## next step

Execute WI-0031 Slice 1 (domain contracts and offline selection trace) under a separate
authorization, beginning with the repository-backed language/policy-authority decision, then
deterministic fixtures before code. A recommendation is not an authorization; no implementation
is authorized by this closeout.
