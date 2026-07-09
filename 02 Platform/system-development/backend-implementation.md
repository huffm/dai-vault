---
title: "Backend Implementation Patterns"
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
  - backend
  - dotnet
related:
  - "02 Platform/system-development/architecture-contracts.md"
  - "02 Platform/system-development/testing-strategy.md"
---

# backend implementation patterns

## purpose

Repo-specific patterns for the .NET platform (`dai/platform/dotnet`, primarily
`DevCore.Api`). Seeded thin from verified observations; grows by promotion from work-item
lessons. UX-relevant contract semantics live in [[architecture-contracts]]; this doc holds
implementation mechanics.

## patterns (each verified in shipped code)

1. **Route constraints are the first validation layer.** Agent-run endpoints constrain ids
   with `{agentRunId:guid}` (`DevCore.Api/Controllers/AgentRunsController.cs`), which accepts
   every .NET GUID format. Client-side validation must never be narrower than the route
   constraint it feeds (this was a shipped defect, fixed 2026-07-09, dai `0119d7d`).

2. **Dev inspection endpoints are read-only by doctrine.** `/artifact`, `/prompt-trace`, and
   recent-run listings serve persisted data only; they never trigger model calls or mutate
   state. New dev endpoints follow the same rule or they are not dev endpoints.

3. **CORS dev origins are explicit.** The `spa` policy allows specific localhost origins
   (`https://localhost:4200`, `http://localhost:4201`, …) in `DevCore.Api/Program.cs`.
   Frontend dev servers must run on an allowed origin; extending the list is a deliberate
   edit, not a wildcard.

4. **Exposure boundary is fail-closed end to end.** Paid or dev-only capabilities are gated
   server-side regardless of frontend flags; frontend gates are UX only. Reference:
   `dai/docs/sports-exposure-boundary.md`, backend `AgentRunAccess` policy.

5. **UX-motivated backend changes are proposed first.** If the frontend genuinely needs a
   clearer contract (batch endpoints, richer error shapes, status semantics), the work item
   spec proposes it under "implementation notes" and the change waits for review. No
   unilateral backend edits from UI slices.

## hard boundary

Never change: decision logic, prompt logic, reconciliation behavior, calibration behavior,
confidence doctrine, artifact doctrine, buyer-facing semantics. A work item that needs any of
these is a different class of work item and says so explicitly in its spec.

## what it is not

Not an API reference (source is), not contract doctrine ([[architecture-contracts]] is),
and not yet comprehensive — `status: in-progress` is honest: EF/data-layer and
agent-service patterns are undocumented until a work item touches them.

## deferred decisions

- Data-layer (EF/DevCore.Data) conventions — document with the first schema-touching work item.
- Error-shape standardization (problem-details vs current shapes) — only if a UX contract
  proposal motivates it.

## recommended next slice

None standalone; this doc grows by promotion.
