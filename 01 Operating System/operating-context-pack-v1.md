# dai operating context pack v1

date: 2026-07-06
purpose: load-bearing context for future advanced-model sessions. dense, factual, reusable. supersedes nothing; compresses vault + code state as of 2026-07-06.
volatility: the "posture as of" paragraph in section 1 is the only intentionally volatile part of this document — re-verify its counts and pending-work claims live before relying on them. sections 2-8 are doctrine and change only by recorded decision.
sources: 01 Operating System/glossary.md, 02 Platform/architecture/* and decisions/0001-0009, 03 Niches/sports-analytics/*, 04 Products/sports-v1/*, 05 Research/market/niche-comparison.md, 06 Execution/system-state-synopsis-v1.md, 06 Execution/reports/interim-system-improvement-clinical-audit-handoff-2026-07-06-v1.md.

---

## 1. one-page executive context

dai is a decision-intelligence platform (factory) that produces niche decision products (assembly lines). the platform runs bounded cognitive workers (agents) inside deterministic pipelines; niches are configuration and workflow layered on shared platform machinery. tenants are the boundary for identity, permissions, data isolation, and billing. frontends are thin packaging. stripe is the eventual source of truth for monetized access (doc-only today).

mental model: platform = factory. agents = workers. tenants = businesses/workspaces. niches = assembly lines. frontends = packaging. stripe = truth.

first wedge: sports-analytics (nfl + nba in v1 scope; mlb in the dev slice to exercise the live path). the product is a decision artifact, not a picks feed: one game produces one compact read that compresses 30-45 minutes of manual checking into under 90 seconds. explicitly not a tout, odds board, sportsbook router, or community.

stack (all live): angular sports-app (:4201) -> .net DevCore.Api (:5007) -> fastapi agent-service (:8000, private loopback) -> gpt-4o-mini. sql server runs as docker container devcore-sql. no orchestration layer; AgentRunService runs four explicit steps: retrieve -> analyze -> evaluate -> compose.

governing principle: integrity before settlement. settlement before calibration. calibration before tuning. evidence before optimization. gate 4 (calibration sufficiency) is the master bottleneck and is FALSE on merits (not process). all tuning, model replacement, and buyer performance claims are locked behind it.

posture as of 2026-07-06: system healthy and correctly gated; the scarce resource is settled evidence, not machinery. 279 agent runs total; 59 valid calibration runs (ExclusionReason IS NULL). a 6-run backed_depth divergence cohort (captured 07-06, ~$0.004 total spend) passed qa and awaits settlement after 07-06 finals (~00:30 ET 07-07). first-ever dai-vs-market backed_depth divergence captured (gamePk 823036, MIL@STL). 436 agent-service tests green. next slice: backed-depth divergence settlement/reconciliation v1, then pre-settlement qa script v1.

non-negotiables: do not market the platform first. do not build dashboards without demand. do not automate uncertainty. do not tie identity to a single product. do not assume scale before revenue.

---

## 2. current architecture doctrine

- cognitive protocol runtime (decision 0004): deterministic platform code moves a shared decision artifact through four macro protocols, each with three micro-actions, plus a final synthesize layer. perceive (detect, frame, aim); interrogate (question, probe, verify); discern (weigh, contrast, stress); decide (resolve, position, justify); synthesize (integrate, compose, deliver — operations, not cognition). retrieve is not a cognitive micro-action; the platform owns retrieval, schema, calibration, persistence, evaluation. the model owns interpretation, counter-case, uncertainty.
- tenant/niche split: tenant logic is core platform logic; niche logic is configuration and workflow layered on top. build one factory, many assembly lines. keep platform logic separate from niche logic in code.
- five reusable agent roles: collector, evaluator, synthesizer, compliance, delivery. stable across niches; niche behavior injected via agent profiles/niche config (AgentProfileKey currently stored null — no profile infrastructure yet).
- run pipeline contract: retrieve -> analyze -> evaluate -> compose via ISportsRetriever/ISportsAnalyzer/ISportsEvaluator/ISportsComposer. analyzer confidence is provisional; the evaluator owns final calibrated AggregateConfidence (deterministic dampening/clamps). artifact contract: sports_decision_artifact_v3. postures: play, pass, monitor, wait, compare, avoid. ui label is "Read Stance," never "Pick."
- settlement model: three separated entities — AgentRun+OutputJson (decision-time snapshot), AgentRunOutcome (raw result), AgentRunEvaluation (derived correct/incorrect/inconclusive via RunEvaluator, same transaction). one production write path (AddOutcomeAndEvaluation), idempotent by agentRunId (retry = 409). no poller or background auto-settle exists by design. every settlement write carries full provenance (SourceRef, Notes) per the canonical reconciliation residue contract (decision 0006). settlement refuses lean-vs-prose contradictions (422, writes nothing).
- tool gateway: every tool call routes through ToolGateway.InvokeAsync; fails closed. stations do not choose tools; the model never selects tools or widens access. status ACTIVE-LivePartial: stage-level AllowedProtocolNodes enforced; idempotency caching, cost-class rate limits, tenant-tier enforcement declarative/deferred.
- routing/observability: observedDataRegime stamped on every run (decision 0007, observability only); source-readiness preflight endpoint predicts regime pre-spend (decision 0008); registry-authoritative routing is paid-canary-ready for starter_enriched_market_backed_depth but DEFAULT-OFF (decision 0009). canary enablement is process-scoped only, restored to off after use.
- correlation: X-Agent-Run-Id is the canonical correlation anchor across db and fastapi.

---

## 3. current product/revenue doctrine

- niche ranking (2026-04): sports-analytics first (pain clarity, speed to v1, platform fit; competitors are touts users want to leave). stock = highest ceiling (~$149/mo b2b) but slower. crypto = crowded. kalshi = tam too small.
- sports v1: decision artifact over picks feed (niche decision 0003). one game -> one compact read, under 90 seconds. five signal categories. delivery: email (starter), slack webhook (pro). three-item nav shell: matchup analyzer / history / account; saved reads is a filter inside history (niche decision 0002). thin frontend slice is live; paid-product scope, richer data coverage, delivery quality in progress.
- monetization posture: stripe is the source of truth for monetized access. the future TierGuard queries the .net db (synced by webhook), never stripe's api on the request path. webhook POST /api/webhooks/stripe, signature-verified, no [Authorize]. indicative tiers: free (limited), starter ~$29/mo, pro ~$79/mo. all of this is documentation only — no stripe code exists and none is to be built until billing is a real near-term milestone.
- performance claims: no buyer-facing accuracy or edge claims until gate 5. current evidence would not support one anyway (calibration assessment v3: 57.7% accuracy on 52; 0.75-bucket outperforms 0.80 — inverted; zero demonstrated edge over market consensus in the v8 cohort: 7/7 marketAgreement true).
- revenue precedes scale: no multi-tenant hardening, no dashboards, no growth machinery before a paying subscriber validates the sports loop end to end.

---

## 4. current technical boundaries

- browser boundary is REST only (angular -> .net, msal interceptor planned via entra external id). no websockets, no graphql. progress streaming, if ever needed, is fetch()+ReadableStream sse, and only when real-time progress is a product requirement.
- .net -> fastapi stays plain http on loopback. no grpc, no proto stubs.
- frontends thin and replaceable: collect input, present results, reduce friction, collect payment. no business logic in the frontend. buyer sanitization is currently presentation-only (known limitation).
- auth: AgentRunsController has no [Authorize] (dev bypass Dev:EnableBypassAuth). direction is entra external id federating google/apple/microsoft. auth0/okta rejected (paid dependency).
- known standing risks (accepted, tracked): unauthed sports-app; /dev/artifacts batch runner; live key in agent-service/.env; presentation-only buyer sanitization.
- data/ops facts: devcore sql server is the docker container devcore-sql (not a windows service). platform-api timeout on agent-runs usually means devcore-sql is down (sql is hit before the model). analyze 500 usually means openai 429 quota. calibration reads must filter ExclusionReason IS NULL.
- code standards: lowercase ascii comments; name concrete files/methods when proposing changes; small precise edits over broad rewrites; keep dai and dai-vault boundaries (decisions in dai-vault/02 Platform/decisions or niche decisions; research in 05 Research; product/niche docs in 03/04).

---

## 5. current no-build list

do not build, scaffold, or enable any of the following until the stated unlock:

- tuning, threshold edits, model replacement, buyer performance claims — locked behind gate 4/5.
- stage-3 mutation of the cognitive factory — locked behind gate 4/5.
- registry routing as default — stays DEFAULT-OFF; paid canary runs only, process-scoped, approval-gated.
- stripe code / TierGuard — doc-only until billing is a real near-term milestone.
- azure functions, grpc stubs, vector/olap layers, agent-profile infrastructure — deferred until concretely needed.
- [Authorize] on AgentRunsController, account linking, jit provisioning — deferred with auth slice.
- per-station gateway enforcement, cost-class/tenant-tier gateway controls — deferred.
- background settlement poller / auto-settle — intentionally absent; settlement is explicit and idempotent.
- more protocol/interrogate machinery — the 07-05 whole-platform audit concluded machinery is mature and correctly gated; interrogate is IntegratedDormant stage 2. interrogate statistical-inference idea is deferred, not started.
- sports scope creep: no props, community, odds board, sportsbook execution in v1; no sports beyond nfl/nba/mlb until a paying subscriber validates end to end. wnba is CaptureReadyWithSmallConfig (spread-only) but wnba-for-volume is contraindicated.
- baseline backfill for gate 4 — rejected as unbackfillable. gate edits to pass gate 4 — the gate stays FALSE on merits; passing it by criterion change is gaming.
- standing non-negotiables: no marketing the platform first; no dashboards without demand; no automating uncertainty; no identity tied to a single product; no assuming scale before revenue.

---

## 6. preferred next-slice selection criteria

rank candidate slices by, in order:

1. evidence yield: does it move settled, valid (ExclusionReason IS NULL) runs toward gate 4? settled evidence is the scarce resource; machinery is not.
2. spend: no-spend slices preferred. paid slices are approval-gated, pre-priced, and pre-flighted (source-readiness check; timing windows matter — divergence capture must run ~10:00-13:00 ET before slates exhaust).
3. blast radius: one choke point, small precise edits, byte-identical outputs where the contract says so, default-off for anything routable.
4. verifiability: tdd — full agent-service suite green (test count is volatile; see the section 1 posture paragraph) — plus live verification against real endpoints; no success claims without run evidence.
5. reversibility: canary flags process-scoped and restored; schema changes never mid-cohort.
6. handoff: every slice ends with a continuation-grade handoff brief (dai-agent-handoff canonical 13-section template — required standard).

anti-criteria: adds machinery ahead of evidence; requires a gate edit to look successful; touches locked layers; couples platform and niche logic.

---

## 7. reusable glossary

- tenant: boundary for identity, permissions, data isolation, billing.
- niche: domain-specific inputs, prompts, thresholds, scoring, output shapes layered on the platform (sports, crypto, stock, kalshi).
- agent: bounded cognitive worker filling a role (collector, evaluator, synthesizer, compliance, delivery) inside a deterministic pipeline.
- agent run: one execution producing a decision artifact; snapshot in AgentRun.OutputJson.
- decision artifact: the product unit — one compact structured read per game (sports_decision_artifact_v3).
- posture / read stance: play, pass, monitor, wait, compare, avoid. never called a pick.
- lean / leanSide: the artifact's directional stance on a market side.
- backed_depth: the enriched market-backed data regime (starter_enriched_market_backed_depth route).
- divergence: dai leanSide != market consensus side (marketAgreement false). the only source of edge-over-market signal.
- settlement: writing AgentRunOutcome + AgentRunEvaluation for a run, idempotent, provenance-complete.
- reconciliation residue contract: every settlement write carries SourceRef + Notes (decision 0006).
- exclusion filter: calibration reads count only settled runs with ExclusionReason IS NULL; valid set = 59 as of 2026-07-06 (volatile — grows with settlement; re-verify live before use).
- evidence gates: gate 1 decision integrity (complete); gate 2 settlement (mature); gate 3 calibration eligibility (operational); gate 4 calibration sufficiency (FALSE — master bottleneck; discrimination_hybrid_v1 criterion in pooled_calibration.py); gate 5 evidence-backed optimization (locked).
- data regime: observedDataRegime stamped per run; selectedDataRegime is selection-only (null on live path).
- source-readiness preflight: read-only GET /api/agent-runs/source-readiness predicting regime before model spend.
- tool gateway: fail-closed choke point for all tool calls (ToolGateway.InvokeAsync); ACTIVE-LivePartial.
- cognitive protocol runtime: perceive/interrogate/discern/decide macro protocols (12 micro-actions) + synthesize layer; retrieve is platform work, not cognition.
- X-Agent-Run-Id: canonical correlation id across db and fastapi.
- prompt ledger: development-time prompt/process tracking; zero runtime footprint.

---

## 8. instructions for future agents working on this project

1. read before proposing: latest handoff brief (06 Execution/reports/ + handoffs/current-slice.md), relevant vault doctrine, then the code. reconcile all three before suggesting changes. audits have corrected prior audits (e.g. disagreement n=4, not 0) — verify against live endpoints, not stale docs. treat this document's section 1 posture paragraph as stale until re-verified.
2. respect the gates. gate 4 is FALSE on merits. do not tune, do not edit gates to pass, do not make performance claims. the correct response to a failing gate is more settled evidence or an honest criterion review with the gate staying false.
3. spend is approval-gated. never make paid model calls without explicit approval and a preflight. default posture is hold/no-spend.
4. settlement discipline: settle only from authoritative finals, post-game; idempotent writes with full provenance; never reconcile pre-finals; exclude contradictions rather than patching them.
5. flags stay default-off. any canary you enable, you restore. never leave registry routing on.
6. workspace boundaries: implementation in dai; strategy/decisions/research in dai-vault. do not move files across without explanation. decisions go in 02 Platform/decisions or niche decisions folders.
7. code style: lowercase ascii comments; small precise edits; name concrete files and methods; tdd against the agent-service suite; summarize changes clearly.
8. visual direction for sports-app is settled (gradient system stable) — do not reopen casually.
9. every substantive slice ends with a continuation-grade 13-section handoff brief (dai-agent-handoff template) and an updated slice log.
10. ops shortcuts: devcore-sql is docker; agent-runs timeout = check sql first; analyze 500 = check openai quota; calibration queries filter ExclusionReason IS NULL.
11. honor the non-negotiables in section 1 and the no-build list in section 5. when in doubt, choose the slice that adds settled evidence over the slice that adds machinery.
