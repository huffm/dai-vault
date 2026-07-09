---
title: "Architecture Contracts"
type: "execution-pattern"
date: "2026-07-09"
status: "in-progress"
project: "DAI"
slice: "DAI System Development v1"
repos:
  dai: "unchanged"
  dai-vault: "docs-only"
tags:
  - system-development
  - architecture
  - contracts
related:
  - "02 Platform/system-development/backend-implementation.md"
  - "02 Platform/system-development/design-system/interaction-states.md"
---

# architecture contracts

## purpose

The cross-cutting contracts every work item must respect: where truth lives, where the buyer
boundary sits, how runs are correlated, and what error semantics mean to the UI.

## truth hierarchy

Authority runs top to bottom; lower never overrides higher (from
`06 Execution/agent-slice-workflow-doctrine-v1.md`):

1. observed runtime behavior and tests
2. source code
3. explicit contracts and vault docs
4. slice handoffs
5. generated graphs (graphify) and prior assumptions — navigation evidence only

## buyer boundary

Buyer surfaces consume server-side projections only. `GET /api/agent-runs/{id}/artifact/buyer`
drops internal fields at the API; the internal artifact endpoint is dev tooling. Hiding
internal data client-side is never an acceptable substitute — a buyer contract change is a
backend change. References: `AgentRunsController.cs`, `sports-api.service.ts`
(`getBuyerAgentRunArtifact`).

## correlation

`X-Agent-Run-Id` is the canonical correlation anchor across the platform and agent service;
run ids surfaced in the UI display as first-8-uppercase (`shortRunId`,
`apps/sports-app/src/app/dev-artifact-review/run-load.ts`) but the full GUID is the
identifier everywhere else. Correlation id shape differs between the db and fastapi layers —
treat the header, not any single store's format, as the anchor.

## dto mirroring

TypeScript models mirror API DTOs in
`apps/sports-app/src/app/core/models/agent-run.model.ts`. A contract change lands in both
places within one work item or it is not done. Optional fields are the back-compat mechanism;
absence means "older record", never "error".

## error semantics (as the UI must interpret them)

- `404` on an artifact read = not persisted (yet) → UI treats as *unavailable* (warning tier,
  amber; retryable), not failure.
- Any other error = *failed* (error tier, red; retryable).
- These map onto the status system in [[interaction-states]]. Changing this mapping is a
  contract change, spec'd and reviewed.

## contract-change procedure

1. The work item spec states the UX need and the proposed contract delta under
   "implementation notes".
2. The architecture-contracts lens reviews against this doc and the hard boundary in
   [[backend-implementation]].
3. Only after acceptance does implementation touch the API, DTOs, and mirrors together.

## what it is not

Not an inventory of every endpoint (source is). Not the decision/prompt/calibration doctrine
layer — those are untouchable from this subtree.

## recommended next slice

None standalone; extend when the first UX-motivated contract proposal arrives.
