---
title: "Tool Authorization Risk Disposition: WI-0023 Absent-Enforcement Findings (2026-07-15)"
type: "evidence-report"
date: "2026-07-15"
status: "complete"
project: "DAI"
slice: "WI-0023 Tool Authorization Fitness v1 -- integration review"
repos:
  dai: "read-only topology inspection at 383d7cb (tests-only branch); no runtime fix"
  dai-vault: "docs-only"
tags:
  - security
  - authorization
  - release
  - topology
related:
  - "06 Execution/reports/tool-authorization-fitness-results-v1.md"
  - "06 Execution/reports/rc-equivalence-wi-0023-2026-07-15-v1.md"
  - "06 Execution/plans/v1-concierge-operations-runbook-v1.md"
---

# tool authorization risk disposition: wi-0023 absent-enforcement findings

Topology classification of the two ABSENT authorization findings from WI-0023. **No
runtime fix occurred; this is a classification + Gate-1-check disposition only.**

## aggregate RC disposition: RC_CONDITIONAL_TOPOLOGY_CHECK

The paid, unauthenticated surfaces are safe in the approved single-operator-host V1
topology ONLY IF their loopback binds (and a firewall on the all-interfaces gRPC port)
hold. Those are runtime startup assumptions, not enforced boundaries -> Gate 1 opening
checks must verify them before any paid execution. Neither finding is a release_blocker
(the paid HTTP surfaces bind loopback by default), and neither is fully cleared (safety
depends on a checkable assumption).

## finding 1: competitions.reference

- **exact path:** `GET /api/competitions/{code}/matchup-dates` (+ /teams, /upcoming),
  `SportsReferenceController` (Route `api`), action `GetMatchupDates`.
- **authorization:** NO `[Authorize]` attribute; no fallback auth policy -> ANONYMOUS.
- **provider / spend:** invokes the PaidExternal `schedule.matchup_dates` tool
  (OddsScheduleClient -> the-odds-api.com) through `ToolGateway.InvokeAsync`.
- **amplification bound:** the paid call fires ONLY after four DB validations
  (supported competition, active competition, teamA active, teamB active) -> caller
  input cannot trigger arbitrary paid work; 30m cache bounds repeats.
- **persistence:** none (read).
- **bind / exposure:** DevCore.Api binds `http://localhost:5007` (launchSettings
  default profile; runbook step 5 = `dotnet run` default profile) = LOOPBACK. Not
  reachable outside the operator host when bound loopback.
- **upstream auth:** none required in the path (the anonymous route is the entry).
- **Gate 1 usage:** the drill uses source-readiness + creation, not this reference
  route; not on the drill's critical path.
- **classification: conditional_rc_risk.** Safe in V1 iff :5007 is loopback-bound.

## finding 2: agent-service.surface

- **exact routes:** HTTP :8000 `/api/ping|chat|assist|sports/analyze|echo|debug`
  (paid model call = `/api/sports/analyze`); gRPC `:50051` (Assist).
- **authorization:** none at the service level (no auth dependency on the analyze
  route; verified in `main.py` route surface).
- **bind facts:** runbook step 5 starts `uvicorn main:app --port 8000` with NO
  `--host` -> uvicorn default host `127.0.0.1` = LOOPBACK (main.py documents
  `--host 127.0.0.1`). The gRPC server calls `add_insecure_port("[::]:50051")` = ALL
  INTERFACES -> LAN-reachable absent a host firewall.
- **intended caller:** DevCore.Api (FastApiClient -> localhost:8000) is the sole
  intended caller; CORS (`allow_origins` localhost:4200) is NOT server-side
  authorization.
- **spend / bounds:** each analyze = a paid model call; 30s model timeout + 1500
  completion-token cap; cost log per call; NO idempotency.
- **Gate 1 usage:** the drill's generation step calls agent-service on :8000 (loopback).
- **classification: conditional_rc_risk.** The paid HTTP surface is loopback-safe; the
  gRPC :50051 all-interfaces bind needs a firewall/network-profile check.

## why not release_blocker

release_blocker requires that an untrusted EXTERNAL caller can trigger paid/write work
in the intended V1 topology with no effective upstream boundary. Here both paid HTTP
surfaces bind LOOPBACK by the runbook's own startup commands, so an external caller
cannot reach them on the operator host. The residual risk is (a) an operator starting a
service with a non-loopback `--host`/profile, and (b) the gRPC :50051 all-interfaces
bind on a host without an inbound firewall -- both reducible to operator-controlled,
Gate-1-verifiable conditions. That is conditional_rc_risk, not a blocker.

## Gate 1 opening-check requirements (added by reference; historical Gate 0 report NOT rewritten)

Before any paid execution on 2026-07-17, the operator must verify and STOP if unproven:
1. DevCore.Api listens on 127.0.0.1:5007 only (no 0.0.0.0 / non-loopback listener).
2. agent-service HTTP listens on 127.0.0.1:8000 only.
3. No external publication of :5007 / :8000 / :50051 (no port forward, tunnel, or
   reverse proxy); host firewall/network profile blocks inbound :50051.
4. Host is the operator-controlled machine (not a shared/exposed host).
Evidence method: `Get-NetTCPConnection -State Listen` + process/owner check + firewall
profile check (socket/process/config evidence -- never invoke a paid route to test).

## cloud-stage implications

Both findings are HARD blockers for any hosted/multi-instance deployment (WI-0014 +
PH-06 Amber): the reference route needs authentication or an authenticated upstream,
and the agent-service surface needs service-level auth (mTLS / token) and loopback or
private-network binding. Not reducible to a topology check once off the single host.

## corrective owner

PH-06 Amber (route + service authentication enforcement) + WI-0014 (cloud gate). No
corrective work is authorized here. **No runtime fix occurred.**
