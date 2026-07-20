---
title: "Capability-Selection Deterministic Plan Building Slice 4 Closeout v1 (2026-07-20)"
type: "evidence-report"
date: "2026-07-20"
status: "complete"
project: "DAI"
slice: "WI-0031 Slice 4: deterministic ranking and bounded plan building"
repos:
  dai: "code+tests (DevCore.Domain capability-selection; branch wi/0031-deterministic-ranking-and-plan-building)"
  dai-vault: "docs-only"
tags:
  - system-development
  - platform
  - capability-selection
  - implementation
related:
  - "02 Platform/system-development/work-items/WI-0031-model-assisted-capability-recommendation-and-tool-selection-v1.md"
  - "02 Platform/architecture/capability-recommendation-and-tool-selection-standard-v1.md"
---

# capability-selection deterministic plan building slice 4 closeout v1

## purpose

WI-0031 Slice 4: the pure, offline, deterministic decision layer between resolved tool
recommendations (Slices 1-3) and any future execution. It answers -- which candidates
passed every hard rule, how allowed candidates rank, which appear in a small proposed
plan, which remain blocked or authorization-pending, and why -- and produces a plan FOR
REVIEW that executes nothing and grants no permission. Offline implementation only; no
model/gateway/network/db/execution calls; local commits only (dai `f926484`), NOT pushed.

## hard rule vs score (the core distinction)

A **hard rule** is a yes/no requirement; if it fails, no score can rescue the candidate.
A **score** only ranks candidates that already passed every hard rule -- it never creates
eligibility. Mapping onto the Slice-2 `ResolutionResult`:

- a shadow candidate (a required check denied/unavailable) -> `blocked`; never scored.
- an accessible candidate with any unevaluated required dimension -> `authorization_pending`
  (fail closed: **not evaluated != allowed**); never scored, never planned.
- an accessible candidate whose required dimensions were all evaluated -> **eligible**;
  scored, ranked, and eligible for the bounded plan.

## why an unevaluated rule is not permission

An unevaluated required permission check is authorization-pending, never allowed. The layer
inherits the WI-0031 binding rule (`NotEvaluated != Allowed`) and applies it at plan time:
a pending candidate can never enter the plan and can never be scored, so a missing check
can never be silently treated as approval.

## what the weight profile controls

A named, versioned `WeightProfile` weights a **closed** set of known score-component names
to rank eligible candidates only. Validation (deterministic, ordered errors) rejects
duplicate component names, unknown component names, missing required components, and NaN/
Infinity/invalid weights. Absent candidate facts contribute the unfavorable floor (0.0);
an unknown-named candidate component is ignored (never favorable). One fixture-backed
profile is delivered (`deterministic-plan-ranking/1.0`); weights are never derived from
outcomes or auto-modified. The profile name and version appear in the plan so the result
is reproducible.

## what the proposed plan contains

A bounded, deterministic ordered list of `ProposedStep`s (rank, capability id, tool id,
score, scored components). Maximum steps must be >= 0 (zero yields an honest empty plan;
negative is rejected). No duplicate tool step (first-ranked wins). Ranking = score
descending, then stable identity tie-break (capability id, then tool id, ascending
ordinal). Blocked and inaccessible recommendations stay in the trace but outside the plan.
The plan carries no credentials, request payloads, execution arguments, callbacks,
delegates, endpoints, or any runtime invocation mechanism, and an all-false authority
ledger.

## why the plan cannot execute / how the Tool Gateway stays the authority

The plan is data for review. It has no invocation mechanism and every authority boolean is
false. Plan status is one of `planned_not_authorized` / `authorization_pending` /
`blocked` -- never approved/authorized/executable/ready_to_run. The plan records
`runtimePermissionAuthority = tool_gateway_runtime_permission_authority`: the Tool Gateway
(DevCore.Api) remains the runtime permission authority and MUST be re-checked if a future
execution slice is ever authorized. This layer creates no second permission authority and
calls no tool.

## architecture and boundaries

Implemented in the existing WI-0031 capability-selection area
(`DevCore.Domain/CapabilitySelection/DeterministicPlanBuilder.cs`), reusing the Slice-2/3
contracts (`ResolutionResult`, `ResolvedCandidate`, `ScoreComponent`, `ResolutionContext`)
and the existing canonical-serialization convention (System.Text.Json camelCase, enum
names, declaration-order properties, pre-ordered collections -> byte-identical output). No
sports/market-contrast logic entered the platform domain; no Tool Gateway call, no DI,
endpoint, CLI, service, scheduler, persistence, telemetry, model call, or runtime link to
WI-0034/WI-0035.

## verification

20 required invariants proven (hard-rule-blocks-high-score; eligible-outranks-denied;
unevaluated-is-pending; unknown/absent-never-favorable; invalid/duplicate/unknown-weights
rejected; input-order-invariant output; identity tie-break; zero-limit empty plan; max
enforced + negative rejected; duplicate-tool dedup; blocked/inaccessible stay in trace;
profile-version visible; byte-identical repeats; no-authority; no-execution; tenant
isolation; Slice-3-composition determinism) plus a fixed-seed 400-iteration corpus proving
reorder-determinism and every invariant. Full DevCore.Api.Tests **1409 passed / 0 failed**
(was 1394, +15); DevCore.Domain build **0 warnings** (only pre-existing NU1903 package
advisory elsewhere; 0 csproj changes on the branch); fresh-process determinism confirmed
across two independent test processes; `git diff --check` clean; secret/machine-path/
authority scans clean; strict snapshot 24 WIs / 0 warnings; protected/classified drift
byte-identical.

## next step

Independent review + integration of the local `wi/0031-deterministic-ranking-and-plan-building`
branches. WI-0031 Slice 5 (execution telemetry / offline evaluation) and Slice 6 (governed
pilot) remain deferred. A proposed plan authorizes no execution, capture, or screening.
