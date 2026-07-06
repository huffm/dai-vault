---
title: "Highest-Leverage Slice Discovery + Interrogate Machinery Audit v1"
type: "report"
date: "2026-07-05"
status: "complete -- no-spend audit; ranked next slices; binding constraint = Gate 4 (calibration sufficiency)"
project: "DAI"
slice: "Highest-Leverage Slice Discovery + Interrogate Machinery Audit"
repos:
  dai: "unchanged (c6d4f43) -- read-only audit"
  dai-vault: "docs-only (this report + handoff)"
tags:
  - architecture
  - cognitive-factory
  - interrogate
  - calibration
  - planning
related:
  - "02 Platform/architecture/cognitive-factory/cognitive-factory-runtime-activation-readiness-v1.md"
  - "02 Platform/architecture/governance/evidence-readiness-gates-v1.md"
  - "06 Execution/reports/backed-depth-divergence-capture-plan-2026-07-05-v1.md"
  - "06 Execution/reports/reconciliation-last-cohort-2026-07-05-v1.md"
---

# Highest-Leverage Slice Discovery + Interrogate Machinery Audit v1

## 1-3. objective

No-spend architecture audit: determine the highest-value next implementation slice, with special attention to
whether the Cognitive Protocol machinery (esp. Interrogate) is implemented / dormant / documented / missing.
Evidence-cited (file:line), no implementation.

## 4. repo state

- dai: `c6d4f43`, 0 ahead / 0 behind; dirty only on pre-existing `platform/dotnet/DevCore.Data/DevCore.Data.csproj`
  (CRLF phantom, empty diff, intentionally untouched).
- dai-vault: `2dcb724`, 1 ahead / 0 behind (prior divergence-capture docs commit, unpushed); untracked
  `06 Execution/system-state-synopsis-v1.md` (pre-existing, not this slice).

## 5-6. inspected

Docs: `01 Operating System/{product-factory-model,principles,decision-rules}.md`; `02 Platform/architecture/
cognitive-factory/*` (runtime-activation-readiness, observability-surface, configuration-bound-control,
read-only-execution, deferred-runtime-decisions-ledger); `02 Platform/architecture/governance/
evidence-readiness-gates-v1.md`; `02 Platform/agents/*`; recent calibration/reconciliation reports.
Code (5 parallel read-only audits): `platform/dotnet/DevCore.Api/{AgentRuns,Protocols,Tools,Identity,Diagnostics,
Controllers}/*`; `services/agent-service/app/services/{sports_analyzer,pooled_calibration,registry_prompt_canary}.py`;
`apps/sports-app/src/app/analyzer/*`.

## 7. Cognitive Protocol maturity matrix

Maturity is judged two ways: (a) contribution to the **persisted live artifact**, and (b) the dedicated **C#
station-runner** subsystem. Key fact: the live artifact's cognitive protocol is **model-produced** (11 of 12
micro-actions emitted by the single analyze model call, parsed + persisted) plus a thin deterministic C# builder;
the elaborate `Protocols/*` "Cognitive Factory" is a separate, dormant subsystem NOT on the paid path.

| Station | Live artifact | C# station-runner | Runtime-wired (paid path) | Evidence |
|---|---|---|---|---|
| Perceive | **Live** (model) | Scaffolded | via model seed -> builder | detect/frame/aim model-emitted; `sports_analyzer.py:418-431`, `CognitiveProtocolBuilder.cs:48-52`. Runners `PerceiveSignalIntake.cs` tests-only |
| Interrogate | **Live** (hybrid) | IntegratedDormant | question/verify=model; **probe=deterministic C#** | `CognitiveProtocolBuilder.cs:54-58,103-209`; `ProbeRequest.cs`. `ProtocolNodeRunner` runs probe only from dev endpoint |
| Discern | **Live** (model) | Scaffolded | weigh/contrast/stress model-emitted | `sports_analyzer.py:436-439`; `DiscernStationRunner.cs` NOT DI-registered ("do not activate") |
| Decide | **Live** (model + clamp) | none live | resolve/position/justify model; confidence overridden by evaluator | `sports_analyzer.py:441-445`; `SportsComposer.cs:70,85` |
| Synthesize | **Live** (constants) | Scaffolded | integrate/compose/deliver = static strings | `CognitiveProtocolBuilder.cs:27-32,74-78`; real assembly is `SportsComposer.Compose` |

Separate C# "Cognitive Factory" subsystem (NONE on the paid path): `ProtocolRegistry` (15 static station cards,
"no run-path code reads it yet" `ProtocolRegistry.cs:5-7`) = Scaffolded-declarative; `ProtocolNodeRunner`
(probe-only, dev/test caller only) = IntegratedDormant; `DiscernStationRunner`/`PerceiveSignalIntake` = Scaffolded;
`ProbeRefresh*` chain = IntegratedDormant. Diagnostics report `CognitiveRuntimeActivated:false`,
`ActivationStage:2 (Read-Only Execution)`.

## 8. Interrogate deep audit (Question / Probe / Verify)

- **Question** (`interrogate.question`) -- **Live-thin.** Model-emitted string (`CognitiveProtocolBuilder.cs:55`);
  no dedicated runner, no structured output, no tool/gateway call, diagnostic-only (no confidence effect).
- **Verify** (`interrogate.verify`) -- **Live-thin.** Same shape as Question (`cs:57`); model string, no structured
  output, no runtime station path.
- **Probe** (`interrogate.probe`) -- **Live + Mature.** Deterministic C# (`BuildProbe`, `cs:103-141`), structured
  `ProbeRequest` (SignalKey/Reason/Priority/ConfidenceEffect, `ProbeRequest.cs:52-57`), on the real compose path,
  `RuntimeMaturity.Mature`. Retrieve-nothing by contract ("Probe may recommend investigation; it does not
  retrieve", `ProbeRequest.cs:1-19`). Descriptive-only confidence effect (changes no rule).

**ProbeRefresh chain (the downstream "refresh source without changing the decision" machinery) = IntegratedDormant:**
- `ProbeRefreshExecutor.cs:93-235` -- retrieve-ONLY: invokes exactly one candidate retrieve tool via `IToolGateway`
  always at `platform.retrieve`; "merges NOTHING, persists NOTHING, re-runs no station" (`:23-24`). Proven by tests
  (`ProbeRefreshExecutorTests.cs:58/76`). 5 supported tools.
- Full chain (probe->decision->authz->execute->perceive->discern->decide->synthesize->mergePlan->review->dryRun->
  audit) stops at **dry-run + audit**; there is NO apply/writer -- "artifact mutation structurally impossible (no
  writer exists)"; confidence/posture/lean blocked at four layers.
- Merge contract (`ProbeRefreshArtifactMergeContract.cs`), audit persistence (`ProbeRefreshMergeAuditStore.cs`,
  writes only `ProbeRefreshMergeAudits`), and a tenant-safe read service (`ProbeRefreshMergeAuditReadService.cs`,
  DI-registered) are all fully implemented -- **but the read service has NO controller/HTTP route.**
- **Default-off at every independent switch** (executor `Disabled`; chain `Enabled/AllowGatewayExecution/
  PersistAuditRecord=false`; merge `ArtifactMergeEnabled=false`; mutation flags inert). Critically, the
  `CognitiveFactory:*` config posture gates ONLY the dev Stage-2 endpoint, NOT the internal ProbeRefresh options
  (those are hardcoded `Disabled` in DI) -- so even all `CognitiveFactory:*`=true does not enable the chain.
- **No hidden paid paths:** grep of `Protocols/*` for model/LLM calls = zero. The only external effect is the
  gateway retrieve. The dev Stage-2 endpoint `POST /api/dev/cognitive-factory/probe/audit-only`
  (`env.IsDevelopment()` + gate) runs one deterministic probe, calls no model/tool/db, mutates nothing.

**Answer to "can Interrogate refresh source state without changing the decision?"** -- Yes, by design and today:
the ProbeRefresh chain is retrieve-only / dry-run-only with no writer. It is production-safe to run read-only, but
its Stage-3 (artifact mutation) activation is **blocked by Evidence Readiness Gate 4/5** (not achieved).

## 9. Tool Gateway / orchestration audit

| concern | status | evidence |
|---|---|---|
| Tool Gateway (authz + audit) | **LIVE / enforced** | `ToolGateway.cs:59-72` fail-closed; on every sports run `SportsRetriever.cs`; `Program.cs:176` |
| Tool Gateway cloud contract | **partial (~2/5)** | rate-limit/cost/idempotency/tenant-tier deferred; TenantKey null on retrieve path; telemetry log-only, no durable tool-audit table |
| tool access policy | **implemented** | `ProtocolToolAccessPolicy.cs:47-65` fail-closed |
| generic vs hard-coded pipeline | **hard-coded sports** | single `RunType` (`AgentRunService.cs:30-34`); competition branches; sport-specific tools by deliberate design (`ToolRegistry.cs:33-35`) |
| agent roles (collector/evaluator/synthesizer/compliance/delivery) | **documented-only** | vault `02 Platform/agents/*`; zero code classes |
| rate limit / budget | **missing** | declarative metadata only |
| audit records | **partial/dormant** | merge audit store exists but dormant; no durable tool-invocation audit |
| execution-chain assembly | **dormant** | `ProbeRefreshChainAssembly` registered Disabled; no live caller |
| default-off posture | **implemented** | `CognitiveFactoryActivationOptions` all-false; probe-refresh Disabled |

Biggest machinery gap for "agents-as-workers factory": **no niche/role-parameterized chain assembler** binding
niche -> roles -> stations -> tools. The pieces (live gateway, station manifest, dormant node runner, documented
roles) exist but nothing binds them. This is correctly DEFERRED (no second-niche demand; doctrine anti-scale).

## 10. calibration and measurement audit (the binding constraint)

- **conclusionsAllowed = FALSE.** Binding failing gate = **every confidence bucket n>=15** (`pooled_calibration.py:
  83,98-99`): only 0.75 qualifies (n~61); 0.80 ~15 after v8; 0.70 + legacy-shrink buckets still <15.
- backed_depth registry route: 22 directional reconciled (15 matched / 7 unmatched, matchRate 0.682), **DAI-vs-market
  disagreement n=0** -- the core measurement hole. The v8 7-game cohort had marketAgreement=true on all 7, so 5/7
  is indistinguishable from "follow the favorite" (market-informed regime pulls the model to the favorite).
- Divergence capture (the instrument for the missing dimension) was BLOCKED pre-spend: 2026-07-05 slate exhausted to
  2 non-close-favorite pre-game games; fix = run 10:00-13:00 ET. 0 spend.
- evidenceRichness uniform = 2 (discriminates nothing). Market baselines cover only ~13/59 runs (22%) -- a Gate-4
  coverage gap.
- **Measurement infrastructure is reliable**: /metrics byte-identical across infra changes; Residue Contract v1
  enforced (422 on thin residue); source-readiness + reconcile-precheck spend protection proven.
- **Master bottleneck = Gate 4 (Calibration Sufficiency), NOT ACHIEVED** (`evidence-readiness-gates-v1.md:97-104`).
  Gate 4 blocks: tuning (Gate 4), buyer performance claims + model replacement (Gate 5), AND ProbeRefresh Stage-3
  artifact mutation (Gate 4/5). It is the single constraint gating nearly all downstream value.

## 11. buyer / product audit

- Buyer copy safety = **implemented** (transport-boundary sanitization `BuyerArtifact.cs:62-96`; advertised-strength
  humility cap `AdvertisedStrength.cs:40-52`; frontend never uses pick/lock/edge vocab). Revenue-critical, keep.
- Buyer surface shows: lean (via run result), confidence band, per-signal state, source depth. **Absent from buyer
  surface though computed internally:** market baseline (marketConsensusSide/implied prob), decision freshness,
  settlement/calibration track record.
- Output is "sellable enough for a hands-on manual-validation loop," NOT yet for unattended buyer trust.
- **Stripe/billing = documented-only.** No SDK/checkout/subscription; hardcoded "$79/month" label
  (`account.component.html:70-71`); `model_metering.py:5,15` "NOT billing."

Buyer opportunity classification: market-baseline + freshness surfacing = **trust-critical** (highest latent
packaging win, uses already-computed internals); settlement/track-record visibility = **trust-critical but
Gate-5-adjacent** (a track-record display edges toward a performance claim -> defer); billing = **premature**.

## 12. ranked next slices (scored 1-5: Factory / Measurement / Revenue / Risk / Readiness / Cost)

| # | slice | F | M | R | Risk | Rdy | Cost | total |
|---|---|---|---|---|---|---|---|---|
| 1 | **Paid Backed-Depth Divergence Capture v2 (morning window)** | 2 | 5 | 3 | 4 | 4 | 4 | **22** |
| 2 | **Gate-4 Evidence-Sufficiency + Market-Baseline Coverage Diagnostic (no-spend)** | 2 | 4 | 2 | 3 | 5 | 5 | **21** |
| 3 | Buyer Trust-Surface Packaging: surface market baseline + freshness (factual, Gate-5-safe, no-spend) | 3 | 2 | 5 | 4 | 4 | 5 | **23**\* |
| 4 | Tool Gateway durable tool-invocation audit (Layer-1 hardening, no runtime decision change) | 4 | 2 | 2 | 4 | 3 | 5 | 20 |
| 5 | Niche/role-parameterized chain assembler (factory generalization) | 5 | 1 | 2 | 2 | 1 | 4 | 15 |

\* Slice 3 scores highest arithmetically on revenue+cost, but is **conditionally premature**: surfacing the market
baseline today would honestly show DAI == market (no measured edge), which undercuts a sale and confirms measurement
must come first. Its revenue value is unlocked only AFTER some edge/divergence evidence exists. So it is ranked
below the two measurement slices on sequencing, not on raw score.

## 13. selected next slice

**#1 -- Paid Backed-Depth Divergence Capture v2, executed in the 10:00-13:00 ET window.** It produces the ONE
evidence dimension the platform is missing and cannot obtain any other way -- DAI-vs-market disagreement on the
backed_depth route (currently n=0). That dimension gates everything downstream: without evidence that DAI ever
diverges from (and beats) the market, tuning stays noise (Gate 4), buyer performance claims stay forbidden (Gate
5), ProbeRefresh Stage-3 stays locked (Gate 4/5), and buyer packaging-for-sale stays premature. It is cheap
(<=12 runs, $0.05), fully planned, and approval-gated. The only thing that failed on 2026-07-05 was timing.

**Immediate no-spend action (because #1 is timing-gated and it is currently evening):** run **#2 -- the Gate-4
Evidence-Sufficiency + Market-Baseline Coverage Diagnostic** now. It quantifies the exact per-bucket distance to
Gate 4, investigates why market baselines cover only ~13/59 runs and whether any are backfillable no-spend (free
Gate-4 progress if so), and finalizes the morning capture parameters. High evidence-per-attention; de-risks and
sequences the paid spend.

## 14. runner-up

**#2 -- Gate-4 Evidence-Sufficiency + Market-Baseline Coverage Diagnostic (no-spend).** See above. If the market
baseline coverage gap turns out backfillable from persisted snapshots, this slice alone advances Gate 4 for $0.

## 15. deferred / do NOT build yet

- **More Interrogate / protocol / Cognitive-Factory machinery** -- it is mature and correctly IntegratedDormant;
  its next activation (Stage-3 mutation) is Gate-4/5-locked. Building more is premature and un-actionable.
- **ProbeRefresh Stage-3 artifact-mutation activation** -- Gate 4/5, not achieved.
- **Niche/role-parameterized factory generalization** -- no second-niche demand; doctrine anti-scale.
- **Full Stripe/billing build** -- premature until a sellable (edge-validated) output exists.
- **Buyer settlement/track-record visibility** -- Gate-5-adjacent (performance claim); defer until Gate 4.
- **Any tuning / confidence / prompt / model change** -- Gate 4/5 locked.

## 16. validation performed

Read-only. 5 parallel evidence audits (Cognitive Protocol stations, Interrogate/ProbeRefresh internals, Tool
Gateway/orchestration, calibration/measurement, buyer/factory-doctrine), each citing file:line, cross-checked
against the activation-readiness + evidence-readiness-gates doctrine. No endpoints hit that mutate; no DB writes;
no services started.

## 17. what did not change

No runtime code, prompts, prompt registry recipes, routing, confidence logic, buyer copy, schema/migrations,
reconciliation. No new AgentRuns, no paid model calls, no DB writes, no services started. dai unchanged at
`c6d4f43`. Only artifacts produced: this report + the handoff.
