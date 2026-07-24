---
title: "WI-0023 Tool Authorization Fitness v1 (PH-06 Green subset)"
type: "plan"
date: "2026-07-15"
status: "complete"
project: "DAI"
slice: "WI-0023 Tool Authorization Fitness v1"
repos:
  dai: "tests-only on branch wi/0023-interrogate-probe-tool-authorization-fitness (integrated -- dai/main ff to 3f244c8 per final disposition; front matter corrected 2026-07-24 audit)"
  dai-vault: "docs-only (coordinated branch; integrated per final disposition; front matter corrected 2026-07-24 audit)"
tags:
  - system-development
  - work-item
related:
  - "06 Execution/reports/tool-authorization-fitness-results-v1.md"
  - "06 Execution/plans/protocol-hardening-candidate-specifications-v1.md"
---

# WI-0023 tool authorization fitness v1

## problem  <!-- LITE -->

WI-0022 showed protections exist only at specific seams and that at least one paid
precondition (source readiness before generation) is procedural, not enforced by the
run-creation path. DAI had no authoritative answer to: which executable capabilities
exist, what effects can they cause, and where are authorization boundaries actually
enforced vs merely declared or procedural?

## desired behavior

A deterministic, machine-checkable capability inventory (tools + providers + routes +
procedures) with a versioned declaration contract that separates declared authorization
from enforced authorization, detects drift against the real registry, and assigns
every gap a follow-up owner -- without adding any runtime enforcement.

## affected surfaces

dai (tests only): DevCore.Api.Tests/ToolAuthorizationFitness/{CapabilityDeclarations.cs,
ToolAuthorizationFitnessTests.cs, CapabilitySeamTests.cs};
services/agent-service/tests/test_tool_authorization_fitness.py. dai-vault: this WI,
tool-authorization-fitness-results-v1.md, ready-queue PH-06 state, current-slice handoff.

## non-goals

No runtime authorization enforcement; no policy engine; no generalized IAM/permission
platform; no ToolGateway/ToolRegistry/route/tenant behavior change; no schema/config
change; no credential access; no G-10/R-05; no enforcement-subset pull; no PH-02..PH-05
pull; no follow-up WIs minted.

## acceptance criteria  <!-- LITE -->

WI-0023 section 23 (30 criteria) all evaluated and satisfied: every registered tool
declared exactly once; providers + routes inventoried; every declaration explicit on
tenant scope/permission/spend/write/auth requirement/enforcement/retry/timeout/
idempotency/observability; secrets by class only; unknown/absent never reported as
enforced; procedural labeled procedural; dev bypass labeled by environment; drift
detection covers the registry (new undeclared tool fails); invalid combinations tested;
deterministic output; no production logic duplicated; no production/runtime change;
gaps owned; no permission platform; evidence classes accurate; PH-02..PH-05 unpulled;
RC assumptions unchanged. RESULT: satisfied -- evidence report sections 7-9.

## test plan  <!-- LITE -->

Static harness (drift + schema + invalid combos + determinism) + real-seam tests
(registry discovery, controller auth metadata, gateway denial, python route surface).
RESULT: focused 16/16 C# + 3/3 py; FULL 1278/1278 C# (+16), 459/459 py (+3).

## implementation notes

Inspect reverified at a0ca54d: 10 registered tools (unchanged); the gateway enforces
ONLY the node allowlist (cost class is metadata); SportsReferenceController is
ANONYMOUS and its matchup-dates action triggers a PaidExternal odds call; the
agent-service HTTP surface is unauthenticated on a paid model route. These two ABSENT
findings are pinned by seam tests so any future [Authorize]/auth addition forces a
declaration update. Declarations are static test-support C# (no production csproj
change). Two material findings extend the WI-0022 pattern (seam-level protections,
procedural preconditions).

## docs to update

Evidence report (new), ready-queue PH-06 state, current-slice, this WI.

## verification commands  <!-- LITE -->

dotnet test DevCore.Api.Tests --filter "FullyQualifiedName~ToolAuthorizationFitness";
dotnet test DevCore.Api.Tests; pytest tests; git diff a0ca54d -- <production trees>
(empty); strict planning snapshot.

## risks

Declaration drift vs code as routes/tools change (mitigation: drift + coverage + POST-
count seam tests fail on additions); the two ABSENT findings could be misread as
release-blocking (mitigation: characterized as local-only-safe, cloud-gated -- not RC
issues). Enforcement is explicitly out of scope.

## links  <!-- LITE -->

- work item: WI-0023 (implements PH-06 Green subset; PH-06 marked pulled+active)
- branch: wi/0023-interrogate-probe-tool-authorization-fitness (dai from a0ca54d +
  coordinated dai-vault from 8c24bd9; "BOTH LOCAL-ONLY" superseded 2026-07-24 audit --
  both pushed and integrated per final disposition)
- pr: —
- commits: dai ef8acde (declarations), 0534de1 (harness), 383d7cb (seam tests);
  vault commits recorded at close
- tests: +16 C# / +3 python, all green; full suites green
- verification notes: tool-authorization-fitness-results-v1.md
- docs updated: evidence report, ready queue, current-slice, this WI
- lessons: two ABSENT enforcement findings (anonymous paid route + unauthenticated
  paid service surface) are single-host-safe today but hard cloud blockers

## final disposition (2026-07-15)

**complete + reviewed + integrated.** Integration review audited terminology
(declared_complete vs enforced vs procedural vs absent) and the two ABSENT findings
against the actual V1 topology; one correction commit applied (dai `3f244c8`: refined
competitions.reference + agent-service.surface with exact bind facts -- :5007/:8000
loopback, gRPC :50051 all-interfaces -- and disposition conditional_rc_risk; added a
topology-dependency integrity test; vault `6036897` aligned the evidence report +
added the risk-disposition record). Final totals: 41 capabilities (10 tools / 8
providers / 16 routes / 7 procedures); enforcement 6 enforced / 8 partial / 8
procedural / 2 ABSENT / 3 n/a (distribution unchanged; wording corrected).

Post-correction suites green (1279 C# / 459 py / 17+3 focused). RC-neutrality proven
(empty production diff a0ca54d..3f244c8; project-graph proof; smoke with BIND
verification confirming :5007/:8000 loopback and :50051 all-interfaces). **Aggregate
RC disposition: RC_CONDITIONAL_TOPOLOGY_CHECK** (risk-disposition record); both ABSENT
findings = conditional_rc_risk (loopback-safe, Gate-1-verifiable). dai/main
fast-forwarded a0ca54d -> `3f244c8` (new RC commit, same artifact); vault/main ->
`6036897` + integration record. Both branches pushed + retained. Evidence class:
contract-represented + fixture-proven + integration-proven. **PH-06 GREEN SUBSET
CLOSED; PH-06 Amber (enforcement) NOT pulled; WIP = 0.** Every gap owned (PH-03, PH-05,
PH-06 Amber, R-04, G-10/R-05, WI-0014); zero deferred candidates. PH-02..PH-05
unpulled; G-10 unexecuted; R-05 gated; RC Gate 1 (2026-07-17) remains next, conditional
on the added topology opening checks (Friday input updated to hash 3f244c8).
