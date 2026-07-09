---
title: "Prompt Observability Dashboard v1 -- dev-only Run Anatomy / Prompt Trace (plan)"
type: "plan"
date: "2026-07-09"
status: "PLANNED -- awaiting implementation approval; read-only additive design, no schema change required"
project: "DAI"
slice: "Prompt Observability / Dev Dashboard Design"
related:
  - "06 Execution/reports/v2-accelerated-capture-day1-2026-07-09-v1.md"
  - "04 Products/sports-v1/prompting/prompt-market-context-hardening-v1.md"
  - "06 Execution/patterns/capture-closeout-run-eligibility-rule-v1.md"
---

# prompt observability dashboard v1 -- plan

## objective

give the dev dashboard a "Run Anatomy / Prompt Trace" section that shows everything that went
into a DAI artifact -- prompt route, recipe, ingredients, staged market facts, attribution guard
result, reconciliation outcome -- WITHOUT exposing chain-of-thought and without inventing data
that is not persisted. dev-only observability; buyer surfaces untouched.

## 1. current surfaces found (verified against source 2026-07-09)

**frontend (apps/sports-app, Angular standalone + signals + Tailwind v4, Vitest):**
- dev page exists: `/dev/artifacts` -> `src/app/dev-artifact-review/` -- fail-closed behind
  `devToolsGuard` (`environment.enableDevTools===true`, canMatch; prod=false) + `agentRunAuthGuard`;
  not in nav. already renders: run overview stat grid, signal table, pipeline steps, coverage
  chips, follow-up/availability tables, quality warnings, decision fields, cognitive-protocol
  cards (with provenance badge), raw-artifact `<details>`.
- api service `src/app/sports-api.service.ts`; models `src/app/core/models/agent-run.model.ts`.
- design language: Tailwind v4 tokens in `src/styles.css` (`--app-*` vars via `@theme inline`),
  card pattern `rounded-[1.5rem] card-surface surface-frame p-6 lg:p-8`, badges `.artifact-status--*`
  / `.artifact-chip--*`, stat grid `.artifact-stat`, native `<details>` collapsibles, dark-only theme.
- NONE of promptSource/recipeId/assembledHash/attributionFidelity*/marketConsensus*/exclusionReason/
  settlement fields exist anywhere in the frontend today (grep-verified zero matches).

**backend (.NET DevCore.Api + python agent-service):**
- `GET /api/agent-runs/{id}/artifact` (AgentRunArtifactDto) ALREADY returns the full
  `PromptRouteProvenance` record (all 19 fields incl. promptSource, selectedPromptRecipeId/Version,
  assembledHash, observedDataRegime, selectedPromptPath, attributionStatus/Reason) deserialized from
  the `PromptRouteProvenanceJson` column, plus sourceEnvelopes, sourceDepth, cognitiveProtocol,
  signal tables. the frontend just never displays the provenance block.
- `GET /api/agent-runs/prompt-route-calibration/rows` returns route+market consensus (raw medians,
  bookCount, marketAgreement)+guard status/reason/interpretation+outcome per row (prose-free row
  doctrine -- deliberately no evidence clause, no prose).
- `GET /{id}/evaluation`, `/recent`, `/reconcile-precheck` as known.
- market memory: `MarketSnapshotBatch` (consensusSide, RAW median implied probs, bookCount,
  disagreementRange, marketsPresent) + per-book `MarketBookLine` (bookmaker/marketType/side/price/
  point/providerUpdatedAt), linked by agentRunId -- persisted but NO read endpoint exposes them.
- attribution guard `MarketAttributionFidelity.Evaluate()` is pure, derive-on-read, called only by
  the calibration exporter; its `Evidence` clause (the exact prose clause that triggered the
  status) is computed then DROPPED (prose-free row doctrine, PromptRouteCalibrationExport.cs:106-115).
- interrogate/probe: `CognitiveProtocolBuilder.BuildProbeRequest()` is pure over persisted
  `SignalFollowUps` -- staged probe facts are derivable on read, not persisted separately.

## 2. data already available per run (no backend change needed to show)

outcome summary (via /rows + /evaluation), prompt route block (via /artifact provenance), source
envelopes incl. modelContextSummary staged-fact strings (via /artifact), regime/route era (v2@v2
recipe id => v2 era; promptSource live+observedDataRegime => v1-era live), evidence richness,
source depth by category, cognitive protocol stages, exclusion/supersession state (/rows),
settlement residue (settlementSource/SourceRef/Notes on /rows).

## 3. data missing (exact, verified)

1. **rendered prompt text: NOT persisted anywhere.** live user message is built in
   `build_mlb_user_message` (sports_analyzer.py) and discarded after the model call; only cost
   telemetry is logged. `assembledHash` = sha256 of assembled registry recipe text, populated ONLY
   on registry-canary rows (null on all live-path rows). slot values are not persisted; raw
   starter/market request contexts are not persisted (InputJson = teams+date only). => the UI must
   show "not currently persisted" honestly; see persistence contract below.
2. **guard evidence clause:** computed in `MarketAttributionFidelityResult.Evidence`, dropped at
   the row projection. needed for the staged-consensus-vs-prose-claim comparison (the exact 822877
   use case).
3. **de-vigged home/away probabilities:** not persisted (grep zero). recomputable derive-on-read
   from persisted `MarketSnapshotBatch.MedianHome/AwayImpliedProbability` with the same `devig_pair`
   normalization (p_h/(p_h+p_a)), when both medians present.
4. **per-book market lines:** persisted (`MarketBookLine`) but not exposed by any endpoint.
5. **staged probe facts / interrogate context:** derivable via the pure `BuildProbeRequest` over
   persisted SignalFollowUps; not exposed today.

## 4. smallest read-model addition (additive, read-only, no schema change, no writes)

**one new dev endpoint: `GET /api/agent-runs/{agentRunId}/prompt-trace`** returning
`PromptTraceDto`, tenant-scoped like the rest of the controller, aggregating derive-on-read:
- `route`: the deserialized PromptRouteProvenance (reuse the /artifact projection) + a computed
  `regimeEra` label ("v2" when selectedPromptRecipeId ends ".v2"; "v1"/"live" otherwise).
- `market`: linked MarketSnapshotBatch (consensusSide, raw medians, bookCount, disagreementRange,
  marketsPresent, fetchedAtUtc, captureReason) + COMPUTED devigHome/devigAway (same normalization as
  devig_pair; null unless both medians present) + optional `books[]` (MarketBookLine projection).
- `attribution`: the FULL MarketAttributionFidelityResult -- status, divergenceInterpretation,
  reason, AND evidence -- via the same pure Evaluate() call the exporter makes, on the same
  persisted inputs. this surfaces evidence on a per-run DEV read while the calibration ROW export
  stays prose-free (doctrine boundary preserved: /rows unchanged).
- `interrogate`: `BuildProbeRequest(signalFollowUps)` output (status + signals with reason/
  suggestedToolId/priority/confidenceEffect) -- staged expectations, NOT reasoning.
- `renderedPrompt`: `{ persisted: false, assembledHash: <hash|null>, note: "prompt bytes are not
  persisted; hash present only on registry-route rows" }` -- never fabricated text.
- `reconciliation`: outcome/evaluation/settlement residue/exclusion state (reuse existing
  projections).

why an endpoint and not fatter /rows: /rows is the calibration export with an explicit prose-free
doctrine; per-run anatomy is a different read purpose. one run per request keeps payloads small
and avoids denominator/doctrine contamination.

**optional (deferred, separate approval) persistence contract for rendered prompts:** additive
nullable `PromptSlotValuesJson` column (or dev-scratch JSONL via the existing default-off
`mlb_request_capture`) capturing the SLOT VALUES dict at decision time on registry-route rows only;
combined with recipeId+version+manifest hashes this makes prompt bytes reproducible and
hash-verifiable without storing model-facing text. NOT required for v1 of the dashboard; do not
implement without explicit approval (it is a write-path change).

## 5. ui component changes (dev page only)

new "Run Anatomy" section in `dev-artifact-review.component.html`, attached between Run Overview
and Brief Signal Table (or directly above Raw Artifact), using existing patterns -- card surface,
`.artifact-stat` grid, `.artifact-status--*` badges, `<details>` collapsibles. sub-sections per the
operator spec:
1. **Outcome Summary** -- final score, lean vs result badge (correct/incorrect/inconclusive),
   market aligned/opposed, confidence, gate impact note (valid-denominator membership: active vs
   excluded).
2. **Prompt Route** -- promptSource badge (registry/live), recipe id@version, route key, regime era
   pill (v1 era / v2 era), assembledHash (mono, copy button), fallbackReason if any, routingReason.
3. **Recipe Ingredients** -- source envelopes table (signalKey, status, depth, freshnessSummary,
   modelContextSummary), starter context, market block: consensus side/team, bookCount, raw medians,
   computed de-vig pair, disagreementRange; evidence richness; source depth by category.
4. **Interrogate Context** -- probe request signals table (staged expectations + required
   market-vs-lean acknowledgment presence on v2 rows); NO hidden reasoning.
5. **Rendered Prompt** -- "not currently persisted" empty-state with the recommended persistence
   contract text; assembledHash shown when present; copy button dev-mode only.
6. **Attribution Guard** -- Pass/Unclear/Fail badge (emerald/amber/red), reason code, divergence
   interpretation, and the side-by-side comparison: staged consensus (side+team) vs prose claim
   (the evidence clause highlighted). this makes an 822877-style Unclear explain itself at a glance.
7. **Reconciliation** -- outcome, evaluation, settled timestamp, residue (source/sourceRef/notes),
   SingleMatch/precheck state, exclusion/superseded chips.
8. **Investigation Notes** -- deterministic checklist derived from the data (no model call):
   prose ambiguity (guard Unclear + reason both_market_directions_asserted -> "opponent-as-object
   mention in a market clause"), source mismatch (envelope freshness vs commence), classifier
   ambiguity, market data issue (medians missing/degenerate), identity issue (refs vs teams),
   model contradiction (guard FAIL).

frontend data plumbing: add `getPromptTrace(id)` to `SportsApiService` + `PromptTraceDto` types in
`agent-run.model.ts`; new pure projection helper `run-anatomy.ts` (badge/label/derivation logic)
so it is unit-testable like `signal-table.ts`.

## 6. tests

- .NET (DevCore.Api.Tests): prompt-trace endpoint returns provenance parity with /artifact; guard
  block parity (status/reason/interpretation) with the calibration exporter for the same run;
  evidence populated for a both-directions fixture; devig computed only when both medians present;
  tenant scoping (foreign tenant -> 404); no writes (db row counts unchanged after GET).
- frontend (Vitest, pure-function convention): `run-anatomy.spec.ts` -- era labeling, badge
  mapping, devig formatting, investigation-notes derivation for Pass/Unclear/Fail fixtures
  (incl. the real 822877 shape).
- verification targets: 1 settled v2 run + the Unclear run aa00433e/822877 (after 07-10 settlement).

## 7. risks

- **doctrine boundary:** evidence clause becomes readable on a per-run dev endpoint. mitigation:
  /rows untouched (export stays prose-free); endpoint lives beside the other internal reads and the
  page is fail-closed dev-only; buyer artifact untouched.
- **null-heavy legacy rows:** pre-0007 runs have null provenance; pre-market-persistence runs have
  no snapshot. UI must render explicit "not recorded" states (pattern already exists on the
  protocol provenance badge).
- **assembledHash absence on live rows** could read as a defect; label it "n/a (live route)".
- **scope creep toward prompt persistence:** explicitly deferred; write-path changes need separate
  approval.

## 8. non-goals

no buyer-facing UI, no prompt-text persistence in v1, no schema/migration, no chain-of-thought or
model reasoning exposure, no change to prompt behavior/registry selection/capture/reconcile/buyer
copy, no new heavy frontend dependencies, no protocol-machinery expansion.

## implementation order (when approved)

1. `PromptTraceDto` + endpoint + .NET tests (additive controller read).
2. frontend service method + types + `run-anatomy.ts` projection + spec.
3. template section + styles reusing existing classes.
4. verify against one settled v2 run + 822877; screenshot for the vault closeout
   (06 Execution/reports/prompt-observability-dashboard-v1.md).
