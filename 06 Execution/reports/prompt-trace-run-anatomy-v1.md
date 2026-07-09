---
title: "Prompt Trace / Run Anatomy v1 -- implementation closeout"
type: "report"
date: "2026-07-09"
status: "SHIPPED -- dai 8438cbe pushed; read-only, dev-only, no schema change, no writes"
project: "DAI"
slice: "Prompt Observability -- Run Anatomy v1 implementation"
related:
  - "06 Execution/plans/prompt-observability-dashboard-v1.md"
  - "06 Execution/reports/v2-accelerated-capture-day1-2026-07-09-v1.md"
---

# prompt trace / run anatomy v1 -- closeout

## 1. objective

implement the approved dev-only, read-only prompt-trace surface: expose the route, recipe,
ingredients, staged facts, attribution guard evidence, and reconciliation state behind an agent
run, on the existing dev artifact dashboard. no prompt behavior, capture, settlement, schema,
buyer, or calibration change.

## 2. plan file used

`06 Execution/plans/prompt-observability-dashboard-v1.md` (authoritative; implemented as designed --
one additive endpoint + one dev-page section; slot-values persistence stays deferred/approval-gated).

## 3. files changed (dai `8438cbe`, 12 files, +1601/-4)

backend:
- NEW `platform/dotnet/DevCore.Api/AgentRuns/PromptTrace.cs` -- PromptTraceDto (nested records:
  identity/route/market/ingredients/interrogate/guard/reconciliation/renderedPrompt) +
  `PromptTraceService` (derive-on-read aggregation).
- `Controllers/AgentRunsController.cs` -- GET `{id}/prompt-trace` action (DI-injected service).
- `Program.cs` -- scoped registration.
- `AgentRuns/PromptRouteCalibrationExport.cs` -- `DeserializeDecision` + `MarketAgreementFor`
  widened private->internal so trace and calibration rows share ONE implementation (no drift).
- NEW tests `PromptTraceServiceTests.cs` (9) + `Integration/PromptTraceEndpointTests.cs` (2).

frontend (dev page only):
- NEW `apps/sports-app/src/app/dev-artifact-review/run-anatomy.ts` (+spec, 15 tests) -- pure
  projection helpers (guard badges, era labels, probability/active-runs formatting, deterministic
  investigation-notes checklist).
- `core/models/agent-run.model.ts` -- PromptTrace DTO types (+SourceSignalEnvelopeDto).
- `sports-api.service.ts` -- `getPromptTrace()`.
- `dev-artifact-review.component.ts/.html` -- Run Anatomy section + fail-soft trace load with
  stale-response guard.

## 4. endpoint added

`GET /api/agent-runs/{agentRunId}/prompt-trace` -- tenant-scoped (same auth policy as the
controller), read-only (integration test asserts zero row deltas), 404 without leaking. derives
everything from persisted data: run row + PromptRouteProvenanceJson + latest linked
MarketSnapshotBatch (same selection rule as the calibration exporter, so the two surfaces cannot
disagree) + outcome/evaluation + OutputJson artifact.

## 5. dto fields exposed (summary)

- identity: run id, status, competition, gameDate, provider/gamePk, team refs, scheduled/started/
  completed, exclusionReason, supersededBy.
- route: all 19 provenance fields + promptRouteKey + regimeEra (from CANONICAL
  selectedPromptVersion -- not a recipe-suffix parse), assembledHash.
- market: consensus side+team, bookCount, raw medians, computed de-vig pair + consensus gap
  (pairwise normalization; null unless both medians present), disagreementRange, fetchedAt.
- ingredients: evidenceRichness, sourceEnvelopes (staged fact strings), sourceDepth.
- interrogate: lean vs staged consensus (sides + team refs), marketAgreement (shared exporter
  rule), requiredAcknowledgment (recoverable from v2 recipe metadata), probe request projected to
  WIRE-SAFE strings (the ProbeRequest enums have no json converter; embedding them raw would have
  serialized ints -- caught in review, fixed with explicit mapping + tests).
- guard: status/reason/divergenceInterpretation + the EVIDENCE clause + staged-consensus fields
  for the side-by-side comparison. /rows stays prose-free (doctrine untouched; evidence lives only
  on this per-run dev read).
- reconciliation: outcome, scores, residue (source/sourceRef/notes), evaluation, activeRunCountForGame
  (single-match indicator, matcher's active-selection rule).
- renderedPrompt: `status: "not_persisted"` + assembledHash passthrough + honest note. no text
  invented; no persistence added this slice.

## 6. ui sections added (dev `/dev/artifacts` page, fail-closed devToolsGuard)

Run Anatomy card with: 1 Outcome Summary (eval badge, score, lean, market-vs-lean, gate-denominator
chip) / 2 Prompt Route (source, era pill, recipe@version, route key, observed regime, fallback) /
3 Recipe Ingredients (market stats incl. de-vig pair + collapsible source-envelope staged-fact
table) / 4 Interrogate Context (staged consensus vs model lean, required acknowledgment, collapsible
probe table) / 5 Rendered Prompt ("Not persisted" state + sha256 + copy button) / 6 Attribution
Guard (Pass/Unclear/Fail badge, reason, staged consensus vs prose evidence clause side-by-side) /
7 Reconciliation (outcome, settled at, residue, active-run count) / 8 Investigation Notes
(deterministic checklist: model contradiction / classifier ambiguity / prompt ambiguity / market
data / identity / provenance / excluded / clean). existing tailwind tokens + .artifact-status
badges + native details collapsibles; no new dependencies; buyer routes untouched (grep-verified
zero references outside the dev component/service/model/helper).

## 7. examples verified (live, rebuilt api)

- `aa00433e` / 822877 (UNCLEAR, v2): era v2; guard Unclear/both_market_directions_asserted with
  evidence = "The consensus favors the Rangers, indicating confidence in their performance against
  the Angels." beside staged consensus home/texas-rangers -- the opponent-as-object diagnosis is
  readable at a glance; investigation note explains it. probe serialized as strings
  (requested/sharp_public/high/reduces). UI rendered end-to-end (screenshot taken in session).
- `b32c433e` / 824579 (settled Pass, v1 registry): era v1, guard Pass with evidence, outcome
  away_win / eval correct / residue statsapi + "gamePk 824579 final away 8 home 1", de-vig computed.
- `396f433e` (2026-03 legacy): honest nulls -- era null, route unknown, market null, guard
  Unclear/no_market_consensus, renderedPrompt not_persisted; UI renders explicit "not recorded"
  states + accurate notes (no snapshot / no identity / provenance not recorded).

## 8. tests run

- DevCore.Api.Tests: **1080/1080 pass** (was 1069 before slice; +9 service +2 integration). TDD:
  red observed before each implementation step (missing type, missing route, enum-vs-string).
- sports-app Vitest: **90/90 pass** (+15 run-anatomy). `ng build --configuration development` clean.
- code review (multi-angle + verify) before completion; all findings fixed and re-tested:
  probe enum int-serialization (wire-safe string mapping), falsy-zero activeRunCountForGame
  (@if dropped 0; helper + test), one-sided-median note missed away-only case (fix + test),
  stale trace race on rapid run switch (current-id guard), era-by-suffix -> canonical
  selectedPromptVersion (test), duplicated MarketAgreementFor/DeserializeDecision -> shared
  internal helpers, inline `new` -> DI registration, comment-case convention. refuted: the
  latest-snapshot selection matches the calibration exporter's rule exactly (documented in code).

## 9. known missing data (unchanged from plan; NOT invented)

rendered prompt bytes (never persisted; live message discarded post-call), slot values (would make
registry prompts reproducible -- deferred, approval-gated write-path change), per-book MarketBookLine
read surface (persisted, still unexposed), guard evidence on /rows (deliberate doctrine, not a gap).

## 10. explicit confirmations

- 0 model calls, 0 capture, 0 /reconcile, 823281 untouched, no deliberate-ledger writes.
- 0 db writes (read-only endpoint; integration test asserts no row deltas), no schema/migration.
- no prompt/registry-selection/confidence/scoring/threshold/buyer-copy/calibration change; /rows
  byte-shape untouched (guard evidence NOT added to it).
- .env untouched; registry default OFF; agent-service stayed DOWN all slice.
- v2 measurement rules untouched; no improvement claims made.

## 11. repo and vault status

dai `8438cbe` PUSHED (0 ahead / 0 behind; only residue = the pre-existing empty-diff
DevCore.Data.csproj phantom). dai-vault: this report + current-slice append committed and pushed
this closeout. services: devcore-sql + DevCore.Api :5007 RUNNING (rebuilt binary with the new
endpoint; needed for tomorrow's settlement pass), agent-service DOWN, dev ng server stopped.

## 12. next recommended action

1. (scheduled, 07-10 ~10:14 ET) settle day-1 cohort -> settled readout report -> day-2 capture per
   the cadence runbook -- use the new Run Anatomy view on the settled 822877 row for the readout's
   attribution note.
2. optional follow-ups, each approval-gated: attribution-classifier opponent-as-object suppression
   (TDD vs frozen baseline replay), slot-values persistence contract (makes registry prompts
   hash-verifiably reproducible), MarketBookLine read surface.
