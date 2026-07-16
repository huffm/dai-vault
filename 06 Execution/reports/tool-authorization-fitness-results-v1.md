---
title: "Tool Authorization Fitness Results v1 (WI-0023, 2026-07-15)"
type: "evidence-report"
date: "2026-07-15"
status: "complete"
project: "DAI"
slice: "WI-0023 Tool Authorization Fitness v1 (PH-06 Green subset)"
repos:
  dai: "tests-only branch wi/0023-interrogate-probe-tool-authorization-fitness (NOT integrated)"
  dai-vault: "docs-only (coordinated branch)"
tags:
  - evidence
  - hardening
  - authorization
  - fitness
related:
  - "06 Execution/plans/protocol-hardening-candidate-specifications-v1.md"
  - "02 Platform/system-development/work-items/WI-0023-tool-authorization-fitness.md"
---

# tool authorization fitness results v1

Machine-checkable inventory of DAI's tools, provider integrations, externally
effective API operations, and operational procedures at dai `a0ca54d`. EVIDENCE CLASS
ACHIEVED: **contract-represented + fixture-proven + integration-proven** (registry
discovery, controller route/authorization metadata, gateway denial, python route
surface). NOT achieved: runtime authorization enforcement where none exists, paid-run
observed, new production-observed, operationally proven, commercially validated. **A
declared capability is not an enforced capability; enforcement is never inferred from
a declaration's presence.**

## 1. capability universe (41 total; bounded and discovered from repo evidence)

10 registered tools + 8 provider integrations + 16 externally effective API
operations + 7 operational procedures. Registered-tool count REVERIFIED = 10 (matches
ToolRegistry.Default() exactly, drift-checked by test). Declaration schema 1.0;
declaration set is static test-support data, not loaded by production.

## 2. registered-tool inventory (all node-allowlist enforced; cost class metadata only)

| tool | spend | node | enforcement |
|---|---|---|---|
| schedule.matchup_dates | paid_external | platform.reference | partially_enforced (node allowlist yes; cost no; ALSO reachable via anonymous route -- see 4) |
| market.football/basketball/baseball.spread | paid_external | platform.retrieve | partially_enforced |
| schedule.basketball.rest_context | free_external | platform.retrieve | partially_enforced |
| pitching.mlb.probable_starters | free_external | platform.retrieve | partially_enforced |
| availability.mlb.lineup | free_external | platform.retrieve | partially_enforced |
| fatigue.mlb.bullpen | free_external | platform.retrieve | partially_enforced |
| market.sharp_public.split | free_external | platform.retrieve | partially_enforced |
| analysis.sports.matchup_read | paid_external | platform.analyze | partially_enforced |

## 3. provider-integration inventory (8)

odds-api (paid; OddsApi:ApiKey), mlb-statsapi (free), espn (free), actionnetwork
(free, unofficial + spoofed UA), openai (paid; OPENAI_API_KEY; 30s timeout + 1500
token cap), agent-service (metered internal; localhost, unauthenticated),
sql-database (ConnectionStrings:Sql; sa-password class), entra-identity (fail-closed
policy + dev bypass double-condition). Secret classes/key names only; no values read.

## 4. externally effective route inventory (16) + enforcement matrix

| capability | tenant scope | permission | spend | write | auth req | enforcement |
|---|---|---|---|---|---|---|
| agent-runs.create | tenant_scoped | execute | paid_external | database | authenticated | partially_enforced (dup-guard + gamePk + auth yes; readiness/spend procedural) |
| agent-runs.source-readiness | tenant_scoped | read | paid_external | none | authenticated | enforced (screen caps procedural) |
| agent-runs.reconcile | tenant_scoped | write | none | reconciliation | explicit_named_authorization | partially_enforced (residue/direction/idempotency yes; operator-naming procedural) |
| agent-runs.outcome | tenant_scoped | write | none | reconciliation | explicit_named_authorization | partially_enforced |
| agent-runs.exclude | tenant_scoped | write | none | database | bounded_operator_authorization | partially_enforced |
| agent-runs.near-close-capture | tenant_scoped | write | paid_external | database | bounded_operator_authorization | procedural |
| agent-runs.buyer-projections | tenant_scoped | read | none | none | authenticated | enforced |
| agent-runs.operator-diagnostics | tenant_scoped | read | none | none | authenticated | enforced |
| agent-runs.retrieval | tenant_scoped | read | none | none | authenticated | enforced |
| **competitions.reference** | public_non_tenant | read | **paid_external** | none | authenticated | **absent** (ANONYMOUS route triggers a paid odds call) |
| ai.complete | tenant_scoped | execute | paid_external | database | authenticated | enforced |
| conversations | tenant_scoped | write | none | database | authenticated | enforced |
| health | public_non_tenant | read | none | none | none | not_applicable |
| dev.cognitive-factory | system_internal | read | none | none | none | partially_enforced (anonymous but 404 outside Development) |
| dev.provision | operator_global | administrative | none | administrative | red_lane_authorization | partially_enforced ([Authorize]+ProvisionKey; dev-use procedural) |
| **agent-service.surface** | system_internal | execute | **paid_external** | none | none | **absent** (unauthenticated paid model surface; single-host trust boundary) |

## 5. matrices (summary)

- **tenant scope:** tenant_scoped 11, public_non_tenant 3 (competitions.reference,
  health; + odds via ref), system_internal 3, operator_global 8 (procedures + provision).
- **spend class:** paid_external 13, free_external 6, metered_internal 1, none 21.
- **write class:** none 20, database 6, reconciliation 3, delivery 1 (procedural),
  credential 1 (procedural), deployment 1 (procedural), local_ephemeral 1,
  administrative 1.
- **authorization requirement:** authenticated 9, explicit_named_authorization 3,
  bounded_operator_authorization 5, red_lane_authorization 3, none 3, not_applicable 9.
- **enforcement status:** enforced 6, partially_enforced 8, procedural 8, absent 2,
  not_applicable 3 (+ providers not_applicable). The 2 ABSENT are the material
  findings, dispositioned conditional_rc_risk (section 6 + risk-disposition record).

## 6. procedural-versus-enforced findings (declaration != enforcement)

- CostClass metadata EXISTS on 4 paid tools but is NOT enforced (gateway enforces only
  the node allowlist) -> spend enforcement = procedural/absent. Callers do not consume
  cost class for blocking. Follow-up: PH-06 Amber (enforcement) + R-04.
- Source readiness is a SEPARATE authenticated read; the paid creation path does NOT
  consult it (WI-0022 PF-01 confirmed) -> readiness precondition is procedural.
  Follow-up: PH-03.
- Reconciliation enforces residue/direction/idempotency in code; the OPERATOR-NAMING
  requirement (which games, which day) is procedural (code requires only authenticated).
  Boundary unchanged.
- Delivery/entitlement: NO runtime capability (WI-0022 PF-15) -- procedural contract
  only. Owner PH-05 (NOT READY).
- Two ABSENT enforcement findings, dispositioned by the 2026-07-15 integration
  review as **conditional_rc_risk** (safety rests on a Gate-1-verifiable bind
  assumption, not an enforced boundary):
  (a) **competitions.reference** -- anonymous routes; matchup-dates triggers a
  PaidExternal odds call ONLY after 4 db validations (real active competition + both
  active teams -> no arbitrary amplification) with a 30m cache; DevCore.Api binds
  http://localhost:5007 (LOOPBACK) via launchSettings default profile + runbook
  `dotnet run`, so it is not externally reachable when bound loopback.
  (b) **agent-service.surface** -- /api/sports/analyze is unauthenticated and paid; the
  runbook's `uvicorn main:app --port 8000` binds 127.0.0.1 (uvicorn default; main.py
  documents --host 127.0.0.1) = LOOPBACK, BUT the gRPC Assist server binds `[::]:50051`
  = ALL INTERFACES, so the paid HTTP surface is loopback-safe while the gRPC surface is
  LAN-reachable absent a firewall.
  Both are safe in the approved single-operator-host V1 topology ONLY IF the loopback
  bind (and a firewall blocking inbound :50051) hold -- these are startup assumptions,
  so Gate 1 opening checks must verify them (see the risk-disposition record). Hard
  cloud-stage blockers. Follow-up: PH-06 Amber (route/service auth) + WI-0014.

## 7. invalid-combination results (WI-0023 section 16)

All 20 checks pass against the declaration set (no invalid combination present).
**The harness validates declaration INTEGRITY; it does NOT assert that all
capabilities are adequately enforced** -- a check passes either because a valid
enforced combination exists OR because an unenforced gap is explicitly declared and
assigned an owner. The checks: paid
without observability (0), paid retries unknown (0), writes without idempotency (0),
reconciliation without named auth (0), credential/deployment without red-lane (0),
tenant-owned with unknown scope (0), requirement without enforcement status (0),
external without timeout (0), unbounded paid/write retries (0), dev bypass as
production (0 -- entra evidence cites IsDevelopment()), secret values (0),
delivery-as-database (0), unknown-as-free (0 -- enum has no bare 'free'),
procedural-as-code-enforced (0), public-as-tenant-scoped (0), self-expansion vocab (0),
no owning service (0), sports policy in platform layer (0), enforcement-from-manifest
(0). Drift: a newly registered tool without a declaration, or a stale declaration,
fails the harness; a new controller without a mapping fails the coverage seam test;
the externally-effective POST count is pinned (11).

## 8. static drift-detection + anti-duplication + determinism proof

Drift: `every_registered_tool_has_exactly_one_declaration` compares the declaration
set against the REAL ToolRegistry.Default().All(); the CANONICAL safeguard is route
IDENTITY (the coverage test maps every assembly controller to a declared capability; a
new controller fails), with the POST-action count (11) pinned only as a SUPPLEMENTARY
drift signal. Anti-duplication: the
harness reads the real registry and real controller attributes via reflection -- it
does NOT reimplement registry or route discovery; cost class and auth attributes are
read from production, not restated as logic. Determinism: declaration serialization
double-run byte-equal + SHA-256 recorded to the test output location; ordinal
ordering; static data (no clock, no randomness). Independence: no network, no model,
no database, no secrets.

## 9. test evidence (dai branch @ 383d7cb)

| suite | command | result | validates |
|---|---|---|---|
| focused C# fitness | dotnet test --filter FullyQualifiedName~ToolAuthorizationFitness | 16/16 passed (73ms) | schema, drift, invalid combos, route metadata, gateway seam |
| focused python | pytest tests/test_tool_authorization_fitness.py | 3/3 passed | agent-service route surface, model bounds, pricing metadata |
| FULL C# | dotnet test DevCore.Api.Tests | 1278/1278 (was 1262; +16) | regression |
| FULL python | pytest tests | 459/459 (was 456; +3) | regression |
| vitest | NOT RUN (no Angular touched) | n/a | |

Test-type coverage: declaration schema + static fitness + real production seams
(registry discovery, controller auth attributes, gateway denial, python routes) +
repository-evidence classification for non-deterministic-seam capabilities. Totals are
NOT semantic or operational validation.

## 10. gaps and follow-up ownership

| gap | owner | release-relevant | tenant | buyer | RC impact |
|---|---|---|---|---|---|
| cost-class metadata not enforced | PH-06 Amber + R-04 | no | no | no | none (procedural caps hold V1) |
| anonymous competitions.reference triggers paid odds call | PH-06 Amber + WI-0014 | no (loopback-bound) | partial (public route) | no | conditional_rc_risk: Gate 1 verifies :5007 loopback bind; cloud blocker |
| unauthenticated agent-service paid surface | PH-06 Amber + WI-0014 | no (loopback :8000; grpc :50051 all-ifaces) | yes (service boundary) | no | conditional_rc_risk: Gate 1 verifies :8000 loopback + firewall on :50051; cloud blocker |
| readiness not enforced on creation | PH-03 | no | no | no | none |
| reconciliation operator-naming procedural | existing reconciliation gate | no | yes | no | none (guards enforced) |
| delivery/entitlement absent | PH-05 (NOT READY) | no | no | yes (future) | none |
| sa-password historical repo exposure | G-10 + R-05 | no | no | no | none |
| prod Entra values unfilled; no CI; manual deploy | WI-0014 (cloud gate) | no | no | no | none |

No gap falls outside the existing candidates; **zero deferred candidate notes needed.**
No queue priority changed.

## 11. limitations

Enforcement facts are pinned at the SEAM level (controller attributes, gateway denial,
route presence). Non-attribute enforcement (e.g. IdentityResolver tenant scoping) is
cited via existing integration suites, not re-exercised here. The two ABSENT findings
are characterized, NOT fixed (enforcement is out of the Green subset). No evidence
class above integration-proven is claimed.

## 12. evidence-class statement

Achieved: contract-represented (all 41), fixture-proven (harness + declaration
contract), integration-proven (registry discovery, controller auth metadata, gateway
denial, agent-service route surface). Explicitly NOT achieved: runtime enforcement of
any currently-unenforced control, paid-run observed, new production-observed,
operationally proven, commercially validated.
