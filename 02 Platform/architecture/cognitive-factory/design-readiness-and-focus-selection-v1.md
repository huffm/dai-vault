# Design Readiness and Focus Selection v1

**date:** 2026-06-08
**status:** focus-selection review. docs only. no runtime code, schema change, prompt change, model-call change, Tool Gateway change, Angular change, endpoint, station activation, artifact mutation, or confidence/posture/lean change.
**scope:** decide where DAI attention goes next. consolidate the 2026-06-07 runtime-adoption-readiness and product-vs-factory deliberations into one decision record, test them against modern engineering practice, DAI doctrine, and the 4x3 / 12-fold cognitive lens, and name one primary and one backup slice. this is the deliberation that `protocol-station-runtime-adoption-readiness-v1.md` scheduled for 2026-06-08.

## skills / guidance used

- local DAI pack (read-only): `dai-grill-with-vault` (read code + vault before asserting; report disagreements), `dai-token-tight` (dense prose), `dai-agent-handoff` (current-slice addendum shape). `dai-write-skill` not used -- no skill authored.
- superpowers, applied manually: `writing-plans` (assessment structured before writing), `verification-before-completion` (git status + change verification below), `systematic-debugging` not triggered -- no code/doc conflict surfaced.

## executive summary

The factory is **design-ready and runtime-deferred by choice, not by gap.** Station cards carry full ownership/maturity/mutation/IO/fallback/forbidden/status metadata across all 15 stations; the shared `ProtocolResultEnvelope` + `ProtocolStepTrace` + diagnostics rollup harvest is landed; the probe-refresh chain is a complete dormant reference loop with audit/readiness/idempotency. None of this is wired to production, and that is correct: there is still no concrete read-only runtime caller with a product or operator need.

This review changes nothing in that verdict. It confirms it and answers the question the readiness note deferred to today: **the largest gap is buyer-visible value, not factory internals.** The sports product promises a signal-scored brief; the signal/quality evidence the factory already produces still lives mostly in internal artifact and calibration surfaces. Building more runtime seams before a buyer can read and trust the brief would deepen an already-asymmetric line and spend effort where money is not.

Primary next slice: **Sports Brief Signal Table v1** (product-facing; uses existing artifact fields; activates nothing).
Backup next slice: **Sports Artifact Productization Review v1** (docs/design scoping of which artifact fields are buyer-ready, if the team wants to de-risk the table's scope before building it).

Runtime adoption -- Synthesize inspection, Perceive read model, Discern runner, probe-refresh activation -- stays deferred. No new deferral is created by this review; deferred-ledger entry 18 is clarified to record that the 2026-06-08 deliberation occurred and reaffirmed product polish.

## current state

Live line (one request -> one shared decision artifact):

```
request
  -> [retrieve: platform plumbing, not a station]  SportsRetriever + typed clients, SignalQuality/FollowUp evaluators
  -> Perceive    {detect, frame, aim}        model-emitted (shared analyze call)
  -> Interrogate {question, probe, verify}    question/verify model-emitted; probe deterministic (BuildProbe)
  -> Discern     {weigh, contrast, stress}    model-emitted; weigh has deterministic grade backbone
  -> Decide      {resolve, position, justify} model-emitted; confidence/position/lean deterministic + clamped
  -> Synthesize  {integrate, compose, deliver} deterministic SportsComposer; no model call
```

Governance/runtime contract state (code-verified):

- `ProtocolRegistry.Default()` populates 15 station cards across the 5 macros; `ProtocolStationCard` encodes ownership, maturity, mutation policy, IO contracts, allowed tools/scripts/memory, fallback, forbidden behavior, status semantics, token budget.
- `ProtocolNodeRunner.ExecuteAsync` supports exactly one station (`interrogate.probe`); every other registered station returns `UnsupportedStation`; unknown ids fail closed (`UnknownStation`).
- `ProtocolResultEnvelope<TStatus>`, `ProtocolStepTrace`, `ProtocolStationDiagnostics`, `ProtocolDiagnosticsRollup`, `ProtocolRegistryValidator` exist and are read-only / dormant.
- Probe-refresh chain (17 seams), `DiscernStationRunner`, `PerceiveSignalIntake`/`Collector`/projections: assembled, tested, DI-unregistered or default-disabled, wired to no pipeline and no endpoint.

Repos at review time: all three clean and even with origin/main. No dirty state.

## what we learned from probe-refresh

Probe-refresh proved a reusable **shape and discipline**, not just a feature: clean cognitive/plumbing boundary (the station emits a `ProbeRequest`; a separate orchestrator decides whether to fetch; cognitive stations never gain tool power), safe-by-default flags, evidence-before-effect (audit contract + idempotent store + tenant-scoped read before any writer), and diagnostics that inspect without executing. The harvest is done: the recurring result envelope and step trace are now shared platform contracts. The residual lesson is a warning -- the **merge half** (plan/review/dry-run/audit, six seams) guards a writer that does not exist and is explicitly deferred. It must not grow further; it is the canonical example of building governance ahead of need.

## what we learned from Perceive intake work

Perceive intake standardization, the analyzer-seed projection, the probe-refresh projection, and the observation collector together prove the platform **can** normalize signals from multiple origins (analyzer-seed/model-emitted, platform-refresh/tool-fetched) into one observation set with source counts and safe rejection. But every one of those is projection/collection only; no production caller consumes the generic result. The lesson: the receive contract is ready, and it should stay dormant until a real consumer needs normalized observations -- generalizing the intake *logic* on one consumer would repeat the speculative-abstraction mistake the rest of the chain avoided.

## what station-card completion changed

It made the station line **legible**, not **executable.** All 15 cards now declare who owns the station, how mature it is, whether it may mutate artifacts, what it reads/writes, and how it behaves under uncertainty/skip/blocked/failure/not-applicable. Diagnostics and the rollup can report all of it read-only. The deciding change is conceptual: there is now a complete, machine-readable manifest a future caller could stand on -- which is precisely why the temptation to mistake metadata for permission is now the largest standing risk. Card completeness lowered the cost of a future safe activation; it did not create the reason for one.

## product-vs-factory fork

This is the load-bearing fork, and it is already resolved in the same direction three times: Factory Line Balance Review v1 flagged the asymmetry; the runtime-adoption-readiness review found no caller; Product vs Factory Deliberation v1 named Sports Brief Signal Table v1. This review independently reaches the same conclusion under a fuller decision matrix. Factory work is justified only when it removes a blocker to artifact quality, safety, observability, cost, or tenant delivery. None of those is the current binding constraint. The binding constraint is whether a buyer can read, trust, and act on the brief -- and that is a product-packaging gap.

## modern software engineering assessment

| practice | current state | reading |
|---|---|---|
| separation of concerns | strong: cognitive stations never hold tool power; retrieve is plumbing; numbers/enums deterministic, text model-bounded | keep |
| bounded contexts | strong on platform side (Protocols namespace, station cards); product/niche logic still leaks into `ProbeRefreshPerceiveIntake` (sports `ToolIds.*`) -- acceptable while single-consumer | watch |
| fail-closed behavior | strong: runner returns `UnsupportedStation`/`UnknownStation`; all enabling/mutation flags default false; protected fields blocked at 4 seams | keep |
| testability | strong: each seam has a test class; envelope/trace/diagnostics are pure value objects | keep |
| observability | partial: rich value-object diagnostics + rollup exist, but no telemetry emission pipeline and no operator surface | gap, not yet binding |
| data contracts | strong and improving: station cards, result envelope, step trace, Perceive observation set | keep |
| feature flags | present as record options (default off); **not bound to config or tenant entitlement** | gap before any activation |
| migration risk | low now (docs/dormant); would rise sharply at FastAPI canonical-field migration and any schema work | defer |
| product validation | **weakest dimension**: no funnel, no proven buyer-readable brief, money not yet the validator | primary gap |
| avoiding speculative abstraction | mostly disciplined; the merge half and one-consumer projections are the speculative edges, correctly parked | hold the line |

Net: engineering health is high; the one practice scoring poorly is **product validation**, which points away from more internals.

## DAI doctrine assessment

- **Platform = factory:** honored. The platform owns orchestration, permissions, diagnostics, contracts; niche logic is quarantined to probe-refresh sports mapping.
- **Agents = workers:** honored. Stations are bounded workers; probe requests, retrieve fetches, perceive receives, discern weighs, decide recommends, synthesize explains, audit records -- the verb boundaries hold in code.
- **Tenants = economic boundaries:** **partial.** Tenant/run/correlation is threaded through the probe-refresh chain and audit store, but no Stripe/entitlement gate binds spend to a tenant tier. This is deferred (ledger entry 11) and is not the next slice -- but it is the doctrine seam most disconnected from "Stripe = truth."
- **Niches = assembly lines:** honored; sports is the only live line.
- **Frontends = packaging:** **this is the lever.** The signal table is packaging of evidence the factory already produces. Sports Brief Signal Table v1 is squarely "frontends as packaging," done thin.
- **Stripe = truth / money is the real validation:** unmet, by design (pre-revenue). The fastest honest path to that truth is a brief worth paying for, not more runtime maturity.

Doctrine verdict: the product slice is the most doctrine-aligned move -- it packages existing factory output for a buyer and moves toward money as validation, without activating uncertainty.

## 4x3 / 12-fold cognitive design assessment

Used as a coherence lens, not a runtime requirement.

- **Is the station map coherent?** Yes. 5 macros x 3 micro-actions = 15 stations, stable in code and vault, no id drift.
- **Does 4x3 fit better than 5-macro?** They are not rivals -- they describe different bands. The **cognitive** work is genuinely 4x3 = 12: Perceive, Interrogate, Discern, Decide, each with three micro-actions; the factory line balance review already counts "twelve cognitive micro-actions, eleven model-emitted plus deterministic probe." **Synthesize is not a fifth cognitive station -- it is the packaging/delivery band** (deterministic, no model, no tool, `SportsComposer`). Read that way, the 12-fold lens *clarifies* the current structure: 12-fold cognition feeding a 3-part synthesis/delivery layer. That maps onto DAI doctrine cleanly (Synthesize = "frontends as packaging" inside the factory). No code refactor toward a forced 12 is warranted; the symbolic correspondence is satisfied by interpretation, not restructuring.
- **Macro vs micro mixed cleanly?** Yes in code (macro on the card, micro as station id). The risk is only in prose, where "station" sometimes means macro and sometimes micro-action.
- **Overloaded stations?** `interrogate.probe` is still the deepest by far (the only runner-supported station, the whole refresh chain). The overload is now *contained*: the reusable shape was harvested, so the depth no longer forces every other station to re-invent plumbing. The remaining probe-specific excess is the merge half, correctly frozen.
- **Missing stations?** None at the cognitive layer. The genuine gaps are **cross-cutting bands**, not stations: a telemetry/observability emitter and a tenant/economic gate -- both deferred, neither a station.
- **Is `interrogate.probe` still overdeveloped relative to the rest?** Yes in absolute depth, but no longer in a way that distorts the shared contract. Continuing to deepen it (merge writer, activation) is explicitly anti-recommended.

## key questions before next slice

**Product validation**
- What is the single buyer-facing brief surface whose absence most blocks a paid signup?
- Can the existing artifact fields (SignalAvailability, follow-up diagnostics, posture, confidence, counter-case, watch-for, what-would-change) carry a trustworthy signal table without any new source?
- What is the smallest brief a real user would archive and pay to keep receiving?

**Runtime architecture**
- Is there a named read-only caller for any station? (current answer: no)
- If a caller appeared, is Synthesize inspection still the safest first target? (current answer: yes)
- Do feature flags need config/tenant binding before any activation? (yes -- precondition, not this slice)

**Cognitive protocol doctrine**
- Do we formally adopt "12-fold cognition + 3-part synthesis band" as the canonical framing, retiring the implication that Synthesize is a peer cognitive station?
- Should prose standardize "macro/station/micro-action" terms to stop the station-word overload?

**Station ownership**
- For any future activation, which owner (platform/model/hybrid) executes, and is the card's owner field authoritative?
- Does Decide ever get a generic protected-field guard separate from the probe-refresh merge planner?

**Tenant/billing boundary**
- When does a tenant/Stripe entitlement gate move from deferred (entry 11) to active, and what triggers it -- a second tenant, or first revenue?
- Does the signal table need any per-tenant gating, or is it uniform v1 packaging? (expected: uniform)

**Tool permissions**
- Does the signal-table slice require any tool or retrieval beyond what the artifact already holds? (must be no)
- Is `platform.retrieve` still the only gateway scope a future station may ever use?

**Artifact quality**
- Which artifact fields are buyer-ready as-is, which need relabeling (proxy/missing/weak), and which must stay internal?
- How do we surface grounded vs ungrounded `signals_used` so the MLB-style overbroad-claim warnings never reach a buyer?

**Diagnostics and observability**
- Is read-only value-object diagnostics enough until activation, or is a telemetry emitter needed first? (enough for now)
- What minimum operator view would a future activation require?

**Launch readiness**
- What is the delivery surface for v1 (email / Slack-webhook vs internal dev artifact view)?
- Is there a funnel/validation plan, or should one precede deeper product build?

## focus options

1. **Product-facing sports brief value** -- package existing artifact evidence into a buyer-readable signal table. Highest buyer value, lowest new-runtime risk, closest to money. **Selected axis.**
2. **Factory/runtime maturity** -- activate or deepen a station runtime path. No caller; deepens asymmetry; against stated bias.
3. **Diagnostics/readiness cleanup** -- more machine-readable readiness reporting. Already adequate (rollup exists); low marginal value now.
4. **Protocol station adoption** -- wire a runner. Blocked: no caller, flags unbound, treats metadata as permission.
5. **Perceive/Discern/Decide/Synthesize balancing** -- harvest more shared shape. Largely done; remaining genericization needs a second consumer.
6. **Tenant/billing/economic boundary work** -- bind Stripe/entitlement. Doctrine-important but premature pre-revenue; defer until a second tenant or first sale.

## decision matrix

Scored 1 (poor) to 5 (strong). "Money proximity" = how directly it moves toward Stripe-as-truth. "Reversible" = how cheaply undone.

| candidate slice | focus axis | buyer value | money proximity | safety/low risk | effort (5=small) | reversible | doctrine fit | unblocks future | total |
|---|---|---|---|---|---|---|---|---|---|
| **Sports Brief Signal Table v1** | product | 5 | 5 | 4 | 3 | 4 | 5 | 3 | **29** |
| Sports Artifact Productization Review v1 | product (docs) | 4 | 4 | 5 | 5 | 5 | 5 | 3 | **31** |
| Product/Funnel Validation Plan v1 | product (docs) | 4 | 5 | 5 | 4 | 5 | 4 | 2 | **29** |
| Synthesize Runtime Inspection v1 | factory | 1 | 1 | 4 | 3 | 4 | 2 | 3 | **18** |
| Perceive Signal Intake Read Model v1 | factory | 1 | 1 | 3 | 3 | 4 | 2 | 3 | **17** |
| Protocol Station Runtime Readiness Diagnostics v1 | diagnostics | 1 | 1 | 5 | 4 | 5 | 3 | 2 | **21** |

Note: the docs-first product options score marginally higher on the raw matrix because they are cheaper and safer, but they do not themselves ship buyer value -- they scope it. The matrix is a tie-break input, not the decision. The decision weights **buyer value delivered**, where Sports Brief Signal Table v1 leads. The productization review is therefore the **backup/de-risk** option, not the primary.

## primary next slice recommendation

**Sports Brief Signal Table v1.**

Turn the signal/quality evidence the artifact already carries into a compact buyer-facing brief surface: per signal category, a grounded / missing / weak / unavailable / proxy-backed state; one short flag phrase (not raw diagnostics); confidence kept as supporting context with partial-evidence caveats visible; posture rendered as "Read Stance", not "Pick"; counter-case, watch-for, and what-would-change preserved concisely; grounded vs ungrounded `signals_used` made obvious.

Why: it is the only candidate that directly increases paid-user artifact usefulness, it consumes existing factory output instead of inventing runtime, it follows the product doctrine that the signal table is the product, and it moves toward money as the real validator. It activates no station, mutates no artifact, changes no confidence/posture/lean, adds no source, and expands no tool power.

Hard scope guardrails for that slice: no new signals or retrieval sources; no confidence/posture/lean change; no artifact mutation or merge writer; no station activation; no probe tool-power expansion; no FastAPI prompt, model-call, Tool Gateway, schema, or Angular change beyond the minimum packaging surface the slice explicitly approves.

## backup next slice

**Sports Artifact Productization Review v1.**

A docs/design scoping pass that audits exactly which current artifact fields are buyer-ready, which need relabeling (proxy/missing/weak), and which must stay internal -- producing the field-by-field spec the Signal Table slice would build against. Choose this if the team wants to lock the buyer-facing field contract before writing packaging code, or if the Signal Table scope feels under-specified. It is cheaper, fully reversible, and strictly product-forward.

(If, and only if, that review surfaces a named read-only caller that existing artifact/diagnostic surfaces cannot satisfy, the runtime backup remains **Synthesize Runtime Inspection v1** -- inspection-only, no model/tool, no artifact mutation, no delivery change. No such caller exists today.)

## anti-goals

- Do not activate any station; do not treat station-card metadata or status semantics as execution permission.
- Do not build the merge writer, a rollback executor, or any artifact mutation path.
- Do not deepen `interrogate.probe` or probe-refresh; do not bind chain flags to config.
- Do not expand probe tool power or grant any cognitive station tool access; gateway stays `platform.retrieve` only.
- Do not change confidence, posture, or lean without calibration proof.
- Do not split the single analyze call into per-station model calls.
- Do not generalize `ProbeRefreshPerceiveIntake` / DiscernReweigh / DecideRecommendation logic on one consumer.
- Do not add a station endpoint, telemetry emitter, tenant/Stripe gate, FastAPI prompt change, schema migration, Angular change, MCP, pgvector, Functions, or Kubernetes work as part of the next slice.
- Do not let this focus-selection review become a runtime implementation slice.

## what remains deferred

Reaffirmed from `deferred-runtime-decisions-ledger-v1.md`; this review revisits none of them as resolved:

- direct Interrogate -> Perceive self-invocation (entry 1) -- forbidden.
- executor gateway activation (entry 2) -- default-off, `platform.retrieve` only.
- artifact merge writer (entry 3) -- parked until evidence + product need.
- confidence/posture/lean mutation (entry 4) -- calibration-gated.
- Decide application / Synthesize surfacing (entries 5, 6) -- recommend/preview only.
- tenant/Stripe economic gate (entry 11) -- threaded, not gated.
- calibration-driven threshold changes (entry 12) -- evidence-gated.
- probe-refresh chain activation (entry 13) -- dormant; waits on 1-3 + flags + telemetry + operator review + economic gate + product approval.
- genericization of probe-refresh seams (entry 14) -- shape harvested, logic parked.
- Perceive signal intake adoption (entry 15) -- contract/projections dormant.
- Discern station runner adoption (entry 16) -- groundwork dormant.
- station-card completion as non-permission (entry 17) -- metadata complete, not executable.
- runtime station adoption before a concrete caller (entry 18) -- deferred; clarified by this review (see below).

## references reviewed

- `<DAI_VAULT_ROOT>/02 Platform/architecture/cognitive-factory/factory-line-balance-v1.md`
- `<DAI_VAULT_ROOT>/02 Platform/architecture/cognitive-factory/factory-line-balance-review-v1.md`
- `<DAI_VAULT_ROOT>/02 Platform/architecture/cognitive-factory/protocol-station-runtime-adoption-readiness-v1.md`
- `<DAI_VAULT_ROOT>/02 Platform/architecture/cognitive-factory/probe-refresh-chain-activation-readiness-v1.md`
- `<DAI_VAULT_ROOT>/02 Platform/architecture/cognitive-factory/deferred-runtime-decisions-ledger-v1.md`
- `<DAI_VAULT_ROOT>/04 Products/sports-v1/product-vs-factory-deliberation-v1.md`
- `<DAI_VAULT_ROOT>/06 Execution/handoffs/current-slice.md`
- code: `<DAI_REPO_ROOT>/platform/dotnet/DevCore.Api/Protocols/ProtocolRegistry.cs` (15 cards, 5 macros)
- code: `<DAI_REPO_ROOT>/platform/dotnet/DevCore.Api/Protocols/ProtocolNodeRunner.cs` (executes only `interrogate.probe`; others `UnsupportedStation`)
- code: `<DAI_REPO_ROOT>/platform/dotnet/DevCore.Api/Protocols/` (`ProtocolStationCard`, `ProtocolResultEnvelope`, `ProtocolStepTrace`, `ProtocolStationDiagnostics`, `ProtocolDiagnosticsRollup`, `DiscernStationRunner`, `PerceiveSignalIntake`, probe-refresh chain)

## verification performed for this note

- docs-only change.
- no runtime files changed.
- no schema/migration files changed.
- no exact local machine paths added; placeholders used throughout.
- ASCII check for this note passed.
- git status / diff --check results recorded in the slice handoff addendum.
