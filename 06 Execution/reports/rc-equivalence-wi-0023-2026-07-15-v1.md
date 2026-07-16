---
title: "RC Equivalence Record: WI-0023 Integration (2026-07-15)"
type: "evidence-report"
date: "2026-07-15"
status: "complete"
project: "DAI"
slice: "WI-0023 Tool Authorization Fitness v1 -- integration"
repos:
  dai: "tests-only fast-forward a0ca54d -> 3f244c8 (RC-equivalent)"
  dai-vault: "docs-only"
tags:
  - release
  - evidence
  - hardening
related:
  - "06 Execution/reports/rc-equivalence-wi-0022-2026-07-15-v1.md"
  - "06 Execution/reports/tool-authorization-risk-disposition-2026-07-15-v1.md"
  - "06 Execution/reports/rc-gate0-readiness-2026-07-15-v1.md"
---

# rc equivalence record: wi-0023 integration

**Old repository RC commit: `a0ca54d`. New repository RC commit: `3f244c8`
(dai/main after the WI-0023 fast-forward).** Supersedes by reference the RC hash in the
WI-0022 RC-equivalence record (rc-equivalence-wi-0022) and the Gate 0 readiness report
(neither is rewritten). The release candidate ARTIFACT is unchanged: tests-and-fixtures
only.

## equivalence evidence

1. **Tests-only change:** 4 commits (ef8acde, 0534de1, 383d7cb, 3f244c8) touching only
   DevCore.Api.Tests/ToolAuthorizationFitness/ + services/agent-service/tests/.
2. **Empty production-source diff:** `git diff a0ca54d..3f244c8` over DevCore.Api,
   DevCore.Data, DevCore.Domain, DevCore.AiClient, agent-service app/main.py, apps/,
   scripts/ = EMPTY.
3. **Project-graph proof:** DevCore.Api.Tests is IsPackable=false and referenced by no
   production project; python app imports no test module; declarations are static test
   data, not loaded at startup.
4. **Runtime smoke + BIND verification (reviewed branch, 2026-07-15):** docker
   devcore-sql Up; API /health 200; agent-service /api/ping 200; zero runs/provider
   calls. Bind evidence (Get-NetTCPConnection): :5007 -> 127.0.0.1 + ::1 (loopback);
   :8000 -> 127.0.0.1 (loopback); :50051 -> :: (all interfaces). Authoritative shutdown
   -> all ports free, docker down (cold-to-cold).
5. **Gate 1 assumptions unchanged** (startup commands, bind requirements, config names,
   source-readiness + creation routes, duplicate protection, metering, brief exports,
   shutdown, hard caps, Stripe test-mode dependency, gamePks 824766/824737).
6. **Test evidence at 3f244c8:** FULL C# 1279/1279; FULL python 459/459; focused fitness
   17/17 + 3/3.

## risk disposition

**RC_CONDITIONAL_TOPOLOGY_CHECK** (see tool-authorization-risk-disposition-2026-07-15-v1).
Friday's RC Gate 1 is executable AND conditional on the added topology opening checks
(loopback binds for :5007/:8000; firewall on the all-interfaces gRPC :50051). No
release_blocker; no runtime fix.

## rollback point

`a0ca54d` (retained branch wi/0022 head + origin history). Rollback = re-freezing main
at a0ca54d; no data/schema/config change either way.

## boundary

Authorizes only the Friday opening-check hash update to `3f244c8` plus the added
topology checks. RC Gate 1 still needs its own drill-day authorization; a passing
verdict authorizes no commercial, cloud, multisport, or PH-06 Amber enforcement work.
