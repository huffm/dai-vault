---
title: "SLICE PROMPT: Durable Cost Evidence v1 (paste-ready)"
type: "handoff"
date: "2026-07-06"
status: "QUEUED -- execute only after the 2026-07-06 backed_depth cohort is settled and no cohort is in flight"
project: "DAI"
related:
  - "06 Execution/reports/pre-settlement-qa-script-and-cohort-discipline-handoff-2026-07-06-v1.md"
  - "02 Platform/architecture/current-agent-run-contract.md"
---

# slice prompt: durable cost evidence v1

paste-ready prompt for a future implementation agent. authored 2026-07-06 while the
settlement lane was blocked on finals. DO NOT RUN while any capture cohort is
unreconciled.

pre-authored context the implementation agent inherits (verified read-only 2026-07-06):
- metering already exists and is pure: `services/agent-service/app/services/model_metering.py`
  (`build_call_metadata` -> model, inputTokens, outputTokens, totalTokens, usageAvailable,
  latencyMs, status, finishReason, requestId, estimatedInputCost, estimatedOutputCost,
  estimatedTotalCost, currency, pricingSource, costEstimateVersion). explicitly labelled
  internal manufacturing-cost telemetry, NOT billing; stripe remains revenue truth.
- the gap: `sports_analyzer.py` emits that record to the `devcore.cost` LOGGER only
  (`_log_call_metadata`, fail-safe, on both ok and error paths). it never enters the
  analyzer response, the AgentRun row, or /rows. cost facts are therefore hand-copied
  into vault reports today.
- platform side: AgentRun already carries DurationMs and PromptRouteProvenanceJson
  (single-json-column precedent for bounded additive evidence); calibration surface is
  PromptRouteCalibrationRow via PromptRouteCalibrationExport (trailing-optional additive
  fields are the established pattern; the aggregate metrics calculator ignores them).

```text
SLICE: Durable Cost Evidence v1
Mode: implementation slice.
  GENERATION_BEHAVIOR_CHANGE=false, PAID_CALLS=false (default; any paid test call needs
  explicit approval first), NO prompt/routing/confidence/gate/model changes, NO
  buyer-facing output change, NO reconciliation, NO new captures, NO registry flag
  changes, additive persistence/read-model ONLY, PUSH only after verification.
HARD PRECONDITION: do not run mid-cohort. verify via strict preflight or /rows that no
  captured cohort is awaiting settlement (the 2026-07-06 backed_depth cohort must be
  settled first). if any cohort is in flight, STOP and report.

OBJECTIVE
Persist the per-run model cost evidence that model_metering.py already computes durably
at generation time, and expose it read-only on the calibration rows surface (/rows), so
evidence-acquisition planning (gate-4 projection, capture cadence) can use queryable
cost facts instead of hand-recorded numbers. cost-per-run and cost-per-evidence-unit
become derivable from /rows alone.

NON-GOALS (forbidden in this slice)
- tuning, threshold edits, model replacement
- prompt changes, routing changes, gate edits
- buyer performance claims or any buyer-facing surface change
- stripe/billing implementation (metering is cost-of-goods telemetry, not billing;
  stripe remains revenue truth)
- dashboard work
- scheduler/background jobs
- direct db mutation outside normal application persistence (no raw sql)
- changing settlement behavior or existing artifact semantics

PROPOSED MINIMAL FIELD SET (verify against existing conventions before finalizing)
the existing metering vocabulary in model_metering.py is authoritative where it exists;
prefer its names over inventing new ones. reconcile this candidate set against it:
  - modelName            (metering calls it `model` -- pick one, stay consistent)
  - inputTokens, outputTokens, totalTokens        (exist in metering)
  - estimatedCostUsd     (metering has estimatedTotalCost + currency -- prefer reusing)
  - costPricingVersion   (metering has costEstimateVersion + pricingSource -- prefer reusing)
  - costRecordedUtc      (new; when the evidence was persisted)
  - costSource           (new; e.g. "model_metering_v1" -- provenance of the estimate)
persistence shape: prefer ONE bounded additive json column on AgentRun (precedent:
PromptRouteProvenanceJson) over many scalar columns, unless existing conventions say
otherwise; /rows then projects discrete camelCase fields from it (precedent: the
provenance header parse in PromptRouteCalibrationExport.Shape).

PERSISTENCE STRATEGY (required)
- additive migration only; no destructive schema change, no column rewrite.
- no data rewrite/backfill unless explicitly approved (historical runs stay null --
  that is honest: their costs were never durably captured).
- null-safe everywhere: every consumer must treat absent cost evidence as null, never
  as zero cost.
- existing consumers remain compatible: /metrics byte-identical; /rows additive only.

GENERATION-TIME CAPTURE (required)
find the existing model-call metering point (_log_call_metadata in sports_analyzer.py)
and carry the SAME record into the analyzer response contract additively, then persist
it on the AgentRun row in the platform's existing run-completion write. one model call
per run stays one model call; the cost log line stays (log + durable are complements).
fail-safe discipline is preserved: a missing usage object must never fail generation or
persistence -- it persists a record with usageAvailable=false and null costs.

READ-MODEL EXPOSURE (required)
expose the durable cost fields read-only on /rows (PromptRouteCalibrationRow) as
trailing-optional additive fields, following the established additive pattern. shape
discipline: existing fields unchanged; /metrics output byte-identical; existing tests
updated only where additive fields legitimately extend fixtures.

SUPERSEDEDBY RIDER: DEFERRED.
durable cost evidence is the slice. the supersededBy /rows field remains a known
read-model gap and does NOT ride along unless the latest handoff at execution time
explicitly orders it. do not expand scope.

TESTS (tdd; write failing tests first)
- agent-service: analyzer response carries the cost record (ok path + error path +
  missing-usage path); existing suite stays green.
- platform: persistence test -- a completed run row carries the cost evidence; a
  historical/legacy run (null column) round-trips null-safely.
- platform: /rows exposes the cost fields; /metrics unchanged (byte/shape regression
  test where available).
- run both relevant suites (dotnet test for DevCore.Api.Tests; pytest for
  agent-service) and record counts before/after.

VERIFICATION (required before commit)
- baseline repo state recorded (dai + dai-vault: branch, head, dirty, ahead/behind).
- migration applied + rolled forward cleanly on the local db; schema diff is additive only.
- endpoint verification against the live local api: a new dev run persists cost
  evidence; a pre-slice run returns nulls; /metrics byte-identical before/after.
- 0 paid model calls unless a live-path proof was explicitly approved beforehand.
- no registry flag left enabled; no cohort in flight at any point.
- final git diff inspected file-by-file before committing; nothing unrelated rides along.

PUSH INSTRUCTIONS
- commit implementation in dai; commit docs/handoff updates in dai-vault.
- git fetch origin, then git status -sb, in each repo.
- push only if the branch is main and NOT behind origin/main. if behind: STOP and
  report the divergence; do not rebase, do not force push.
- do not push unrelated dirty/untracked files (csproj phantom, manifest json, synopsis).
- after push, confirm local equals origin/main except known intentional untracked files.

FINAL HANDOFF (required)
continuation-grade 13-section handoff in dai-vault/06 Execution/reports/:
objective / outcome / repo state before-after / services used / work performed /
files changed / db writes + side effects / paid calls + cost / validation proof /
what did not change / open issues / recommended next slice / suggested next prompt.
```
