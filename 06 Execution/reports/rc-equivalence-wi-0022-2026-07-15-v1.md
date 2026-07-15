---
title: "RC Equivalence Record: WI-0022 Integration (2026-07-15)"
type: "evidence-report"
date: "2026-07-15"
status: "complete"
project: "DAI"
slice: "WI-0022 Representative Protocol Failure Corpus v1 -- integration"
repos:
  dai: "tests-only fast-forward 85a8831 -> a0ca54d (RC-equivalent)"
  dai-vault: "docs-only"
tags:
  - release
  - evidence
  - hardening
related:
  - "06 Execution/reports/rc-gate0-readiness-2026-07-15-v1.md"
  - "06 Execution/reports/protocol-failure-corpus-results-v1.md"
---

# rc equivalence record: wi-0022 integration

**Old repository RC commit: `85a8831`. New repository RC commit: `a0ca54d`
(dai/main after the WI-0022 fast-forward).** This record supersedes the hash
references in the Gate 0 readiness report (rc-gate0-readiness-2026-07-15-v1.md,
which is NOT rewritten): Friday's RC Gate 1 opening check (runbook 1.1) now expects
**dai main == origin/main == `a0ca54d`** and dai-vault at the current main. The RC
remains the SAME release candidate: the production artifact is unchanged.

## equivalence evidence

1. **Tests-and-fixtures-only change:** 4 commits (e348310, d35de5c, f057a39,
   a0ca54d) adding exactly 4 files under DevCore.Api.Tests/ProtocolFailureCorpus/
   and services/agent-service/tests/. No other path touched.
2. **Empty production-source diff:** `git diff 85a8831..a0ca54d` over
   platform/dotnet/DevCore.Api, DevCore.Data, DevCore.Domain,
   services/agent-service/app, services/agent-service/main.py, apps/, scripts/,
   compose.smoke.yaml = EMPTY.
3. **Project-graph proof:** DevCore.Api.csproj references only Domain/Data/AiClient
   (its sole "Tests" mention is the InternalsVisibleTo attribute -- a compile-time
   visibility grant, not a code reference); the test project is IsPackable=false and
   referenced by nothing in the production graph; python app/ imports no tests; no
   production publish path includes test assemblies.
4. **Runtime smoke from the reviewed branch (2026-07-15):** docker devcore-sql Up;
   DevCore.Api health 200 {"status":"ok"} on 5007; agent-service /api/ping 200 on
   8000; NO runs created, NO provider calls; authoritative shutdown -> all ports
   free, docker down (cold-to-cold).
5. **Gate 1 assumptions unchanged:** commands, routes, configuration presence,
   duplicate-409 protection, metering pricing coverage, brief/recap export
   determinism, shutdown script, hard caps (2 calls/2 runs), Stripe dependency
   status (operator-action-required), and drill gamePks 824766/824737 (still 0 runs)
   are all untouched by construction (no production change).
6. **Test evidence at a0ca54d:** FULL C# 1262/1262; FULL python 456/456; focused
   corpus 27/27 + 3/3.

## rollback point

`85a8831` (retained branch wi/0013-pilot-ops-hardening and origin history). Rollback
= re-freezing main at 85a8831; no data, schema, or config change is involved either
way.

## boundary

This equivalence record authorizes nothing beyond the hash update for Friday's
opening check: RC Gate 1 still requires its own drill-day authorization; a passing
verdict still authorizes no commercial, cloud, multisport, or further engineering
action.
