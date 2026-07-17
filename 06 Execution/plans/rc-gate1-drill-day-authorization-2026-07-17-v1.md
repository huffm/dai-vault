---
title: "RC Gate 1 Drill-Day Authorization v1 (2026-07-17) -- DRAFT"
type: "plan"
date: "2026-07-17"
status: "blocked"
project: "DAI"
slice: "RC Gate 1 Same-Morning Readiness Refresh 2026-07-17"
repos:
  dai: "unchanged"
  dai-vault: "docs-only"
tags:
  - release
  - operations
  - authorization
related:
  - "06 Execution/reports/rc-gate1-readiness-refresh-2026-07-17-v1.md"
  - "06 Execution/plans/v1-rc-drill-package-v1.md"
  - "06 Execution/plans/v1-concierge-operations-runbook-v1.md"
  - "06 Execution/reports/tool-authorization-risk-disposition-2026-07-15-v1.md"
---

# rc gate 1 drill-day authorization v1 (2026-07-17) -- DRAFT

> **DRAFT -- REQUIRES OPERATOR SIGN-OFF. THIS DOCUMENT AUTHORIZES NOTHING UNTIL SIGNED.**
> Produced by the same-morning readiness refresh; the refresh verified evidence and
> drafted this authorization but cannot approve it. The operator is the authorization
> boundary. **As drafted, the candidate list is INVALID** (see candidate status below)
> and must be amended before sign-off.

## scope when signed

Authorizes ONLY the RC Gate 1 pregame drill, executed per the corrected local
`command-checklist.md` sections 1-16 (workspace `rc-drill-2026-07-17/`), the runbook
sections 1-4, 7, 8, and the drill package steps 1-8. No reconciliation, no Gate 2
activity, no outreach, no commercial action.

## repository identity (re-verify live at drill open)

- dai `main == origin/main == c6166e2de9238b4109beb6a975fd2f830447ef13` (RC-equivalent
  chain 85a8831 -> a0ca54d -> 3f244c8 -> 876b73a -> c6166e2; final move adjudicated
  RC_EQUIVALENT by the 2026-07-17 refresh)
- dai-vault main `5ff51f2` or a reviewed successor (documentation state)
- working-tree exceptions exactly: dai csproj phantom; vault graph.json + the 2
  documented untracked files

## candidate status (2026-07-17 17:54Z, statsapi)

- gamePk 824766 (game 1): **In Progress** -- pregame window elapsed; NOT eligible
- gamePk 824737 (game 2): Scheduled 23:10Z
- the doubleheader experiment is **DEFERRED** (not an RC implementation failure; never
  run only one member as the experiment)
- **operator must name fallback candidates**: one or two ordinary eligible MLB games
  from the remaining pregame slate; whether 824737 may serve as a single ordinary
  candidate is an operator decision

## hard caps (global, absolute)

- paid model calls: max 2 | created sports runs: max 2 | duplicate re-POST: max 1
- paid external-source attempts: max 6 GLOBAL; readiness screens: max 2 per gamePk
  (incl. any recovery/R9 re-screen); every failed, invalid-key, or forced-outage
  attempt counts against both caps
- reconciliation 0 | real delivery 0 | outreach 0 | production-code changes 0
- cap breach = stop condition = drill FAIL

## mandatory topology opening stop-gate (after startup, before any external/paid step)

- :5007 and :8000 observed loopback-only; :50051 owning PID + full executable path
  resolved; owner must be the agent-service `.venv` python
- KNOWN RISK: an enabled Public-profile, all-port inbound Allow firewall rule (display
  name "Python") is scoped to a system Python executable outside the DAI workspace;
  re-query the live rule, compare with the :50051 owner, **STOP on coverage or ambiguity**
- no external publication of 5007/8000/50051; operator attestation: no tunnel, reverse
  proxy, remote forwarding, or router port forwarding; operator-controlled host
- STOP if any item is absent, ambiguous, or unproven; no firewall change authorized

## stripe

Refresh classification: **MISSING** -> entitlement criterion capped at CONDITIONAL
PASS. The Stripe test-mode dry-run remains separately gated and defaults to NOT
AUTHORIZED regardless of link availability.

## binding execution order

1 repository/authorization verification; 2 cold-state verification; 3 controlled
startup; 4 live topology verification (stop-gate); 5 configuration presence (no secret
disclosure); 6 public identity/schedule check; 7 initial paid source-readiness screens;
8 forced source-outage and recovery (MUST precede generation); 9 final eligibility
decision; 10 paid generation (canary-first); 11 identity/provenance/metering
verification; 12 brief + markdown verification; 13 simulated delivery only; 14 exactly
one duplicate re-POST expecting 409; 15 shutdown + cold verification; 16 Gate 1 report.

## stop conditions and boundaries

Per the corrected workspace `stop-conditions-checklist.md`. Zero reconciliation, zero
real Stripe transaction, zero real buyer delivery, zero outreach, zero
prompt/model/scoring/schema/production-code change. A passing Gate 1 verdict authorizes
no commercial action; Gate 2 requires its own authorization.

## sign-off (operator only)

- signature: ______  date/time: ______  candidates authorized: ______
- deviations from this draft: ______

> **DRAFT -- REQUIRES OPERATOR SIGN-OFF. UNSIGNED, THIS DOCUMENT AUTHORIZES NOTHING.**
