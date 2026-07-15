---
title: "Cloud Evolution and Multisport Runway v1"
type: "plan"
date: "2026-07-14"
status: "active"
project: "DAI"
slice: "Release Timeline, Architecture Runway, and Multisport Sequence Review v1"
repos:
  dai: "unchanged (evidence at 85a8831)"
  dai-vault: "docs-only"
tags:
  - architecture
  - cloud
  - multisport
  - planning
related:
  - "06 Execution/plans/competition-capability-matrix-v1.md"
  - "06 Execution/plans/v1-to-v2-release-sequence-v1.md"
  - "06 Execution/reports/release-architecture-review-2026-07-14-v1.md"
---

# cloud evolution and multisport runway v1

Provider-neutral stages with one provisional recommendation. Nothing here is
authorized work; every stage sits behind the gates in the release-sequence doc.

## 1. cloud evolution stages

### stage 1 -- local single-host operator environment (CURRENT, V1.0/V1.1)
Topology: one Windows operator host; Docker devcore-sql; DevCore.Api (5007);
agent-service on generation days; sports-app as operator console. Data: SQL Server
container, EF-migrated schema (25+ migrations exist; applied out-of-band). Secrets:
gitignored local files. Identity: dev bypass (Development-only, fail-closed) for the
operator; buyers never touch the system (concierge email). Concurrency: single
process -- the process-local duplicate gate IS valid here. Background jobs: none (by
design; scripts + endpoints). Observability: structured logs + cost log lines +
prompt-trace. Backups: operator-manual (add a docker volume backup step to the runbook
at first paid buyer). Deployment: git + local build. Cost risk: none. EXIT: cloud gate
(>= 2 retained payers OR > 45 op-min/day OR hosted-access requirement).

### stage 2 -- cloud single-instance environment (V1.2 entry)
Topology: ONE instance each of API + agent-service as containers; managed SQL;
same synchronous generation path (no queue -- single instance preserves current
semantics incl. the process-local duplicate gate, which remains VALID at exactly one
API replica, enforced by pinning replicas=1). Data: managed SQL Server-compatible db;
EXPLICIT migration step in the deploy pipeline (repo has migrations; Program.cs does
not Migrate() at boot -- keep it that way, migrate at deploy). Networking: private
service-to-service; TLS at ingress; CORS updated to the real origin; prod frontend
apiBaseUrl replaced. Secrets: managed vault; rotate the odds key + sa password BEFORE
first deploy. Identity: Entra prod registration values filled (fail-loud guard already
exists); operator-only access. Observability: container logs shipped + a durable cost-
log sink + health/readiness probes (health exists; add readiness). Backups: managed db
PITR. Deployment: CI (build + full suites) + pipeline deploy + smoke (health, /brief on
a seeded run) + one-command rollback to previous image + migration rollback script.
Cost risks: managed SQL tier, egress; set budget alerts. EXIT: stage-2 stable for 2+
weeks of pilot operation.

### stage 3 -- production-managed single-region (V1.2 mature)
Adds: environment separation (staging + prod), alerting on error/latency/cost,
backup-restore DRILLED, tenant onboarding procedure (manual, documented), custom
domain + TLS automation, secrets rotation schedule, per-run usage record persisted
(replacing log-only metering) -- prerequisite for real billing hooks. Still replicas=1.
EXIT: buyer count or availability need that a single instance cannot serve.

### stage 4 -- optional multi-instance (NOT approved; gated)
BLOCKED today by: (1) the process-local duplicate gate -- replacement = db-coordinated
enforcement: preferred mechanism a FILTERED UNIQUE INDEX on active-run canonical
identity (TenantKey, Competition, GameDate, normalized matchup or ExternalGameId)
WHERE ExclusionReason IS NULL AND Status IN (pending, completed), with the 409 mapped
from the unique-violation; alternative sp_getapplock around [check+insert]. Either
way: a MIGRATION -- separately authorized (WI-0015). Tests: two-instance concurrent
create harness proving one-run-one-409 against the db mechanism. (2) per-instance
IMemoryCache for starters (WI-0005 recovery assumes one instance) -> distributed cache
or per-request fetch. (3) stdout cost telemetry -> persisted sink (stage 3 does this).
This stage blocks NOTHING before it; do not build for it early.

## 2. recommended first cloud target (provisional, reversible)

**Recommendation: Azure Container Apps + Azure SQL.** Evidence-based fit: the API
Dockerfile header already documents ACA-shaped env config (ConnectionStrings__Sql,
AzureAd__*, APPLICATIONINSIGHTS_CONNECTION_STRING); auth is MSAL/Entra; the db is SQL
Server (Azure SQL is the least-translation move); compose.smoke exists for container
verification; the operator profile is Windows/.NET-native. Compared alternative: **a
single Azure VM lift-and-shift** (docker compose on one VM) -- cheaper and closest to
stage 1, but recreates pet-server ops (patching, manual TLS, no rollback story) and
buys no runway to stage 3; rejected unless ACA cost/complexity surprises in a spike.
No broader vendor survey is warranted -- the stack decides.

## 3. sport qualification ladder (repeatable per competition)

| stage | entry | exit | spend | writes | evidence artifact | stop |
|---|---|---|---|---|---|---|
| 1 contract + adapter design | second-sport gate open | capability descriptor + adapter contracts reviewed | 0 | docs | design doc | doctrine violation |
| 2 deterministic fixtures | 1 done | adapter suites green incl. outcome semantics | 0 | test dbs | suite record | contract gap |
| 3 adversarial identity fixtures | 2 done | same-day repeats, renames, postponements, (college: neutral site) proven | 0 | test dbs | suite record | identity model breaks |
| 4 live read-only source verify | 3 done | schedule + identity + market reads on real data, N days | 0 (free reads; bounded odds calls) | none | readout doc | provider gaps |
| 5 non-paid end-to-end | 4 done | /source-readiness-equivalent + dry-run pipeline w/o model | 0 | none | readout | readiness never eligible |
| 6 bounded paid generation | operator authorization | <= N canary runs, provenance + identity verified per run | capped model+odds | run rows | capture report | guard FAIL / identity mismatch |
| 7 identity + buyer-contract verify | 6 done | /brief + /recap render claim-safe for the sport | 0 | none | contract check | copy unsafe / frame breaks |
| 8 outcome + reconciliation verify | finals + authorization | settlement lands SingleMatch w/ residue; recap correct | 0 | outcomes | settlement report | mismatch |
| 9 repeated cohort | separate authorization | 2+ slate days, 0 hard stops | capped | rows+outcomes | cohort report | stop conditions |
| 10 runbook addendum | 9 done | operator can run the sport from docs alone | 0 | docs | runbook delta | gaps found |
| 11 buyer sample | commercial gate | prospect receives sport sample | <= 1 call | ledger | ledger row | no interest |
| 12 paid pilot | payment | first paid delivery for the sport | pilot caps | product rows | pilot record | commercial stop criteria |

Each stage's authorization is separate; no stage may borrow a prior stage's spend.

## 4. NBA-specific ladder notes (the recommended second sport)

Stage-1 must solve the ONE gating gap the matrix identifies: a stable settlement-grade
event identity provider (statsapi-equivalent -- e.g. NBA stats game ids) feeding
(SourceProvider, ExternalGameId); everything downstream (matcher, guard, recap) then
works unchanged. Evidence contract: rest/schedule (exists via ESPN), market (exists),
lineup/injury depth (new; scope minimally -- readiness policy can declare thin-data
honest postures instead of building every source). Recipes: nba.pregame.* under the
existing registry pattern. Outcome semantics: clean (no ties).

## 5. collegiate prerequisites (own release, never "another code")

(1) NBA ladder complete -- proves the contract generalizes once. (2) Feasibility SPIKE
(read-only, XS-S): provider coverage + identity stability across ~360 NCAAB teams +
market depth sampling on real slates. (3) Outcome-model extension for neutral-site +
tournament advancement (breaks the current two-team home/away winner stack -- known
structural gap). (4) Thin-data-dominant readiness policy (expect no-position to be the
majority posture). (5) Naming normalization strategy (school vs team vs abbreviation
drift). Only then does NCAAB enter the ladder at stage 1.

## 6. architecture fitness checks (evolvability, cheapest effective form)

| check | form | status |
|---|---|---|
| buyer projections carry no numeric confidence | automated tests | EXISTS (WI-0011/12 suites) |
| every configured model has metering coverage | automated test | EXISTS (WI-0013) |
| distinct gamePks never collide | automated tests | EXISTS (guard suite) |
| unknown source state fails closed | automated tests | EXISTS (readiness + recap states) |
| tenant-scoped queries on buyer/run surfaces | integration tests | EXISTS (404 matrices) |
| duplicate prevention valid for deployed topology | pipeline check: assert replicas==1 until WI-0015 ships | ADD at stage 2 |
| no niche code in platform core | static rule (architecture test: platform namespaces must not reference sport classes) | ADD with WI-0016 (the seam only exists then) |
| every competition declares capabilities | contract test over the descriptor | ADD with WI-0016 |
| provider adapters pass identity contract tests | ladder stage 3 suite | ADD per sport |
| no buyer route exposes internal diagnostics | automated payload tests | EXISTS (sentinel tests) |
| deployment artifact reproducible / env rebuildable from config | pipeline check + quarterly rebuild drill | ADD at stage 2 |
| rollback tested | pipeline smoke on previous image | ADD at stage 2 |
Checks NOT adopted (cost > risk now): mutation testing, dependency-graph linting,
runtime architecture monitors.
