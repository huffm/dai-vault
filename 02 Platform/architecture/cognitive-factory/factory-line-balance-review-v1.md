# Factory Line Balance Review v1

**date:** 2026-06-06
**status:** architecture assessment. docs only. no runtime, schema, gateway, prompt, or Angular change. recommends the next implementation slice; activation of the probe-refresh chain remains deferred.
**scope:** the whole cognitive factory line (Perceive, Interrogate, Discern, Decide, Synthesize) measured against the deeply-built `interrogate.probe` refresh path. the question this review answers: having proven a reference pattern on one station loop, where is the line out of balance, and what single slice best strengthens the wider line next.

## skills / guidance used

- local DAI pack (read-only): `dai-grill-with-vault` (read code + vault before asserting; report disagreements rather than smoothing them), `dai-token-tight` (dense prose), `dai-agent-handoff` (current-slice addendum shape). `dai-write-skill` not used -- no skill was authored.
- superpowers: `planning` / `writing-plans` (assessment structure designed before writing), `verification-before-completion` (git status + file-change verification below). `systematic-debugging` not needed -- no repo/doc conflict surfaced.

## executive summary

The factory line is **deep at one station and thin everywhere else.** `interrogate.probe` and its dormant refresh chain now carry seventeen named seams (request -> decision -> authorization -> executor -> perceive intake -> discern re-weigh -> decide recommendation -> synthesize preview -> merge plan -> review -> dry-run -> audit record -> audit store -> audit read -> chain assembly -> chain diagnostics), each a typed `.NET` value-object service with its own test class, plus an activation-readiness review and a deferred-decisions ledger. The other four macro protocols have **no station-specific runtime seam at all**: Perceive, Discern, and Decide are eleven model-emitted micro-actions produced inside one FastAPI analyze call, and Synthesize is deterministic `SportsComposer`. Generic station infrastructure exists but is shallow: `ProtocolNodeRunner.ExecuteAsync` supports exactly one station (`interrogate.probe`); `ProtocolStationCard` v0 encodes ten of the blueprint's eighteen fields; `ProtocolStationDiagnostics` and `ProtocolToolAccessPolicy` are real but read-only and dormant.

The probe-refresh work was the right way to prove a pattern, and it did. The risk now is **overbuilding one station loop** -- adding a merge writer, activation, or more probe-specific depth -- before the proven pattern is harvested into shared platform infrastructure the other four protocols can stand on.

The single highest-leverage next slice is **Generic Station Result Envelope v1**: extract the result-shape that already recurs across every probe-refresh seam (a `Status` enum + `Reason` + optional `ErrorMessage` + an `IsX` convenience bool, plus the `(Step, Reached, Outcome)` chain trace) into one generic envelope every future station returns. It is a structural harvest of a pattern that is well past the rule-of-three, not a speculative generalization; it activates nothing, changes no product behavior, and gives the four thin protocols a consistent result/diagnostic contract to grow into. Backup: **Protocol Diagnostics Rollup v1**.

Probe-refresh should **pause before the merge writer and before activation** (this review finds nothing that changes that). Artifact mutation, confidence/posture/lean mutation, and direct Interrogate -> Perceive self-invocation remain forbidden. Tool power stays under `platform.retrieve`, never on `interrogate.probe`.

## current cognitive factory line

One user request creates one shared decision artifact. Deterministic platform code moves it through the fixed pipeline; the model fills bounded text into pre-scoped fields; numbers and enums stay deterministic.

```
request
  -> [retrieve: platform plumbing, not a station]  SportsRetriever + typed HTTP clients
                                                     SignalQualityEvaluator, SignalFollowUpEvaluator
  -> Perceive   {Detect, Frame, Aim}      model-emitted (shared analyze call)
  -> Interrogate{Question, Probe, Verify}  Question/Verify model-emitted; Probe deterministic
  -> Discern    {Weigh, Contrast, Stress} model-emitted (Weigh has a deterministic backbone)
  -> Decide     {Resolve, Position, Justify} model-emitted; Confidence/Position deterministic + clamped
  -> Synthesize {Integrate, Compose, Deliver} deterministic SportsComposer, no model call
```

Eleven of the twelve cognitive micro-actions are produced by **one** analyze call (`SportsAnalyzerProtocolSeed`); `interrogate.probe` is the single deterministic cognitive station (`CognitiveProtocolBuilder.BuildProbe`); the Synthesize trio is deterministic platform output. Tool governance runs at the **stage** level today (`platform.reference`, `platform.retrieve`, `platform.analyze`) through the Tool Gateway; per-station enforcement is doctrine that becomes honest only after the FastAPI canonical-field migration.

Parallel to that live line sits the **dormant probe-refresh chain** hanging off `interrogate.probe`: a value-object path that proves a refresh could re-enter the read, produce a merge plan, review it, dry-run it, and audit it -- without mutating anything. It is wired into no pipeline and no endpoint.

## current macro protocol map

| macro protocol | live runtime today | model vs platform | dedicated runtime seams beyond the analyze call |
|---|---|---|---|
| Perceive | 3 model-emitted micro-actions in one analyze call | model owns Detect/Frame/Aim text; platform owns the retrieved context blocks they read | none generic; `ProbeRefreshPerceiveIntake` exists but is probe-refresh + sports specific |
| Interrogate | Question/Verify model-emitted; Probe deterministic | model owns Question/Verify; platform owns Probe (`BuildProbe`) | the entire 17-seam refresh chain + `ProtocolNodeRunner` execute path |
| Discern | 3 model-emitted; Weigh has a deterministic grade backbone | platform owns signal grades (`SignalQualityEvaluator`); model owns narrative | `ProbeRefreshDiscernReweigh` (probe-refresh specific) |
| Decide | Resolve/Position/Justify model-emitted; numbers deterministic | platform owns Confidence (`SportsEvaluator`), Position enum clamp, LeanSide clamp | `ProbeRefreshDecideRecommendation` (probe-refresh specific, recommend-only) |
| Synthesize | deterministic `SportsComposer` + `CognitiveProtocolBuilder` | fully platform-owned; model never invents | `ProbeRefreshSynthesizePreview` (preview-only, probe-refresh specific) |

The asymmetry is the headline: Interrogate has a deep dormant loop and a one-station runner execute path; the other four have only their slice of the shared analyze call plus a probe-refresh-specific helper that does not generalize as written.

## probe-refresh as reference implementation

The chain proved a reusable shape, and the discipline held throughout:

- **clean cognitive / plumbing boundary.** `interrogate.probe` only emits a structured `ProbeRequest`. A separate orchestrator (decision -> authorization -> executor) decides whether to fetch. Cognitive stations never gained retrieve-tool power. The executor is the only seam that can touch the gateway, only at `platform.retrieve`, only after authorization.
- **safe-by-default everywhere.** Every enabling flag (`Enabled`, `AllowGatewayExecution`, `PersistAuditRecord`, all four mutation flags, `ProbeRefreshExecutorOptions.Enabled`) defaults false. The mutation flags are inert -- there is no writer to read them.
- **evidence before effect.** Audit contract, idempotent audit store, and tenant-scoped read surface exist before any writer. Protected fields (confidence, posture, lean, artifact version, tenant, run id) are classified forbidden by the merge planner, blocked by review, filtered by dry-run, and surfaced by diagnostics -- one boundary enforced four times.
- **diagnostics without execution.** `ProbeRefreshChainDiagnostics` inspects supplied options/results; it does not run the chain. `ProbeRefreshChainAssembly` returns a `(Step, Reached, Outcome)` trace so "how far did the chain get" is answerable.

This is a genuine reference pattern. The reference is the **shape and discipline**, not the sports-specific payload mapping inside it.

## maturity matrix by macro protocol

Dimensions scored: `mature` (built and exercised), `partial` (exists but thin or dormant), `seed` (doctrine/contract only), `none`.

| dimension | Perceive | Interrogate | Discern | Decide | Synthesize |
|---|---|---|---|---|---|
| canonical station exists (registry card) | mature | mature | mature | mature | mature |
| model-owned vs platform-owned defined | mature | mature | mature | mature | mature |
| analyzer-seed vs completed-protocol boundary clear | mature | mature | mature | mature | mature |
| runtime runner support exists | none | partial (probe only) | none | none | none |
| tool policy support exists | seed (card, unenforced) | partial (retrieve via chain) | seed | seed | seed |
| Tool Gateway path exists if needed | n/a (no tool) | partial (executor, dormant) | n/a | n/a | n/a |
| diagnostics exist | none | mature (chain + station) | none | none | none |
| artifact contribution is clear | mature | mature | mature | mature | mature |
| audit / read support exists if needed | n/a | mature (dormant) | n/a | n/a | n/a |
| mutation / write governance exists if needed | n/a | mature (forbidden by guard) | n/a | partial (recommend-only) | n/a |
| tenant boundary considered | partial (artifact-level) | mature (threaded through chain) | partial | partial | partial |
| cloud / future runtime implications known | mature (cloud plan) | mature | mature | mature | mature |
| calibration dependency known | mature | mature | mature | mature (confidence deferred) | n/a |
| deferred decisions tracked | partial | mature (ledger 13 rows) | partial | mature (entries 4,5) | mature (entry 6) |

Reading: doctrine maturity (cards, ownership, boundary) is uniformly high -- the blueprint and node-specs did that work. **Runtime** maturity is concentrated almost entirely in Interrogate, and within Interrogate almost entirely in the dormant refresh chain.

## maturity matrix by station (where practical)

Only stations with a runtime seam beyond the shared analyze call are listed; the remaining eleven are "model-emitted inside one analyze call, no dedicated seam."

| station | dedicated seam | maturity | note |
|---|---|---|---|
| interrogate.probe | `CognitiveProtocolBuilder.BuildProbe` (live) + `ProtocolNodeRunner.ExecuteAsync` (dormant) + full refresh chain | mature, dormant | the only station with a real execute path |
| perceive (refresh) | `ProbeRefreshPerceiveIntake` | partial | hard-keyed to sports `ToolIds.*` and context types; not generic as written |
| discern (refresh) | `ProbeRefreshDiscernReweigh` | partial | probe-refresh specific assessment |
| decide (refresh) | `ProbeRefreshDecideRecommendation` | partial | recommend-only; mutates nothing |
| synthesize (refresh) | `ProbeRefreshSynthesizePreview` | partial | preview-only; never user-facing |
| (all) | `ProtocolStationDiagnostics`, `ProtocolToolAccessPolicy`, `ProtocolNodeRunner.Resolve` | partial | generic, read-only, dormant; resolve works for all 15 cards, execute for one |

## overbuilt areas

Not "wrong" -- these were correct to build to prove the pattern. They are overbuilt **relative to the rest of the line** and must not grow further before the line catches up.

- **the probe-refresh merge half.** Merge plan, review, dry-run, audit record, audit store, and audit read surface are a full evidence pipeline for a writer that does not exist and is explicitly deferred (ledger entry 3). Six seams guard a mutation that cannot happen. Correct as dormant scaffolding; wrong to extend.
- **per-seam result plumbing duplicated.** Every `ProbeRefresh*` service re-declares the same result shape: a status enum, a `Reason` string, an `ErrorMessage`, and an `IsX` convenience bool. The pattern is repeated fifteen-plus times. That is past the rule-of-three and is the clearest harvest target (see reusable patterns).
- **probe-only execute path on a generic runner.** `ProtocolNodeRunner.ExecuteAsync` is a generic-looking seam that supports a single station and returns `UnsupportedStation` for the other fourteen. The generality is currently notional.

## underbuilt areas

- **Perceive signal intake is not a platform layer.** Retrieval stamps grounded evidence and Perceive consumes it, but there is no reusable "refreshed signal re-enters the read" intake -- `ProbeRefreshPerceiveIntake` is welded to sports tool ids. A second consumer would copy it.
- **no generic station result / diagnostic contract.** Each seam invents its own envelope, so diagnostics, telemetry tags, and "how far did it get" tracing cannot be rolled up uniformly.
- **runner has one real station.** Discern, Decide, and the Synthesize trio have no execute path; the runner cannot drive them.
- **per-station tool enforcement is doctrine only.** Cards carry `allowed_tools`, but enforcement stays at the stage level until the FastAPI canonical migration. The card encodes ten of eighteen blueprint fields (`input_contract`, `output_contract`, `allowed_scripts`, `fallback_behavior`, `forbidden_behavior`, `token_budget`, `allowed_memory_queries`, `artifact_fields_*` deferred -- acknowledged in the card header, not a doc/code conflict).
- **Decide has no guard layer of its own.** Protected-field protection lives inside the probe-refresh merge planner. There is no generic Decide policy guard that would protect confidence/posture/lean for any future write path, not just the probe-refresh one.

## reusable patterns discovered from probe-refresh

These are the harvestable assets. All are **shape**, not sports payload.

1. **station result envelope.** `(Status enum, Reason string, ErrorMessage string?, IsX bool)` -- universal across every seam. The single best generalization candidate.
2. **chain step trace.** `ProbeRefreshChainStepResult(Step, Reached, Outcome)` plus a `List<>` accumulated through the pipeline -- a generic "how far did this protocol run get" diagnostic.
3. **safe-by-default options record.** An options record where every enabling flag defaults false and dangerous flags are inert until a writer exists -- a reusable activation-discipline pattern.
4. **fail-closed payload recovery.** Type-erased payload recovered with `is` pattern checks keyed by id, unknown -> a clean "unsupported / invalid" status rather than a throw.
5. **read-only diagnostics over a dormant seam.** `ProbeRefreshChainDiagnostics` and `ProtocolStationDiagnostics` inspect without executing -- the safe way to make any dormant station legible.
6. **forbidden-field guard, enforced at multiple seams.** One protected-field list enforced by planner + review + dry-run + diagnostics -- the template for any future write governance.

## what should remain probe-refresh specific

- the **sports payload mapping** in `ProbeRefreshPerceiveIntake` (the `ToolIds.*` switch and the `Summarize*` lines). This is niche/competition logic and belongs to the refresh chain, not the platform.
- `ProbeRefreshDiscernReweigh`, `ProbeRefreshDecideRecommendation`, and `ProbeRefreshSynthesizePreview` **as concrete implementations** -- until a second use case appears, generalizing them would be speculative (rule-of-three not met for the re-weigh / recommend / preview *logic*, only for the result *shape*).
- the **merge half** (plan/review/dry-run/audit) stays probe-refresh specific and dormant until a merge writer is genuinely needed.

## what should become generic platform infrastructure

- the **station result envelope** and the **chain step trace** (patterns 1 and 2) -- harvest now; they are past rule-of-three.
- the **read-only diagnostics shape** (pattern 5) -- a generic per-protocol diagnostics rollup is a natural second slice.
- eventually, a **generic Perceive signal-intake layer** (`PerceiveSignalIntake`) -- but only when a second consumer exists; today it would be a one-use abstraction.
- eventually, a **generic Decide policy guard** lifting the forbidden-field discipline out of the merge planner so any write path inherits it.

## deferred items that should remain parked

Reaffirmed from the deferred-runtime-decisions ledger; this review finds no reason to revisit any:

- **direct Interrogate -> Perceive self-invocation** (ledger 1) -- forbidden; the orchestrator triggers refresh, never the cognitive station.
- **executor gateway activation** (ledger 2) -- default-disabled; only `platform.retrieve`.
- **artifact merge writer** (ledger 3) -- no writer; parked until evidence + product need.
- **confidence / posture / lean mutation** (ledger 4) -- forbidden; needs calibration proof.
- **Decide application and Synthesize surfacing** (ledger 5, 6) -- recommendation/preview only.
- **chain activation** (ledger 13) -- dormant; waits on 1-3, flags, telemetry, operator review, tenant/economic gating, and product approval.

## explicit boundary restatements

Per direction, restated so they cannot erode:

- **probe-refresh pauses before the merge writer and before activation.** This review finds nothing that changes that.
- **artifact mutation remains deferred.**
- **confidence / posture / lean mutation remains deferred** (calibration-gated).
- **direct Interrogate -> Perceive self-invocation remains forbidden.**
- **tool power remains under `platform.retrieve`, never on `interrogate.probe`.**
- **the next work strengthens the wider factory line**, not the probe-refresh loop's depth.

## open questions for the architect / founder

Only the ones that could change the next slice:

1. **Factory maturity vs sports product polish.** This is the load-bearing fork. The stated direction is "strengthen the wider factory line," which this review follows. If the real near-term priority is launch polish, the recommendation flips to **Sports Artifact Productization Review v1**. Confirm the axis before the slice starts.
2. **Generalize Perceive signal intake now, or wait for a second consumer?** Recommendation: wait. `ProbeRefreshPerceiveIntake` has one consumer; generalizing on one use case violates the rule-of-three the rest of this chain respected.
3. **Generic StationResult envelope before more station seams?** Recommendation: yes -- it is the prerequisite that keeps the next four protocols from each re-inventing the shape. This is the recommended slice.
4. **Discern re-weigh: generic or probe-specific?** Recommendation: probe-specific until a second use case appears. Harvest the result *shape*, not the re-weigh *logic*.
5. **Minimum product-facing artifact improvement before more runtime depth?** Open. If question 1 resolves toward product, this becomes the scoping question for the backup-flipped slice.
6. **Decide policy guard now or with the merge writer?** Recommendation: defer with the merge writer (ledger 5) unless a non-probe write path appears first.

## recommended next implementation slice

**Primary: Generic Station Result Envelope v1.**

Extract the result shape that already recurs across every probe-refresh seam (`Status` + `Reason` + `ErrorMessage?` + `IsX`) and the chain step trace (`Step, Reached, Outcome`) into one generic, well-tested envelope in `DevCore.Api.Protocols`, and re-express the existing seams against it without changing any behavior. Why this one:

- it is a **harvest, not a guess** -- the pattern is past rule-of-three (fifteen-plus call sites), so it carries no speculative-abstraction risk.
- it **strengthens the wider line**: every future Perceive/Discern/Decide station seam inherits a consistent result + diagnostics + telemetry contract instead of re-inventing one.
- it **activates nothing** -- no gateway call, no writer, no endpoint, no prompt/confidence/posture/schema/Angular change; it is a structural refactor plus a contract.
- it directly **reduces the overbuild asymmetry** by turning duplicated probe-specific plumbing into shared platform infrastructure the thin protocols can stand on.

Scope guardrails for that slice: docs + `.NET` refactor and tests only; no change to the probe-refresh chain's observable behavior; no station logic generalized (only the result/trace shape); the envelope is declarative and adopted seam-by-seam.

**Backup: Protocol Diagnostics Rollup v1.**

A read-only generic diagnostics rollup that unifies the now-fragmenting diagnostics surfaces (`ProbeRefreshChainDiagnostics`, `ProtocolStationDiagnostics`, merge-review telemetry, step traces) into one per-protocol inspection shape. Safe (read-only, dormant), strengthens the wider line, and pairs naturally with the envelope. Chosen over Decide Policy Guard and Discern Station Runner Groundwork because both of those deepen probe-adjacent runtime before the shared contract exists -- the opposite of the stated direction.

## anti-goals

- do **not** build the merge writer, a rollback executor, or any artifact mutation.
- do **not** activate the chain, bind a feature flag to config, or add an HTTP endpoint.
- do **not** generalize `ProbeRefreshPerceiveIntake` / DiscernReweigh / DecideRecommendation logic on a single use case.
- do **not** split the single analyze call into per-station calls (undecided; not required).
- do **not** change confidence rules, the posture enum, or any calibration auto-feedback.
- do **not** touch FastAPI prompts, Pydantic contracts, the Tool Gateway interface, schema/migrations, MCP, pgvector, Azure Functions, Kubernetes, or Angular.
- do **not** wire `interrogate.probe` to a retrieve tool or let any cognitive station gain tool power.

## references

- `02 Platform/architecture/cognitive-factory/protocol-node-specs.md`
- `02 Platform/architecture/cognitive-factory/protocol-station-blueprint-v1.md`
- `02 Platform/architecture/cognitive-factory/probe-refresh-chain-activation-readiness-v1.md`
- `02 Platform/architecture/cognitive-factory/deferred-runtime-decisions-ledger-v1.md`
- `02 Platform/architecture/cloud-tool-runtime-plan.md`
- code: `<DAI_REPO_ROOT>/platform/dotnet/DevCore.Api/Protocols/` (the `ProbeRefresh*` chain, `ProtocolNodeRunner`, `ProtocolStationCard`, `ProtocolStationDiagnostics`, `ProtocolToolAccessPolicy`, `ProtocolRegistry`)
- code: `<DAI_REPO_ROOT>/platform/dotnet/DevCore.Api/AgentRuns/` (`CognitiveProtocolBuilder`, `SportsComposer`, `SportsEvaluator`, `SignalQualityEvaluator`)
