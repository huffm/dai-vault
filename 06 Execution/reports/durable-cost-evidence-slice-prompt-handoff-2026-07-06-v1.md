---
title: "HANDOFF: Durable Cost Evidence v1 Slice Prompt (docs-only) (2026-07-06)"
type: "handoff"
date: "2026-07-06"
status: "COMPLETE -- implementation prompt authored and queued; execution gated post-settlement"
project: "DAI"
slice: "Durable Cost Evidence v1 -- Slice Prompt Document"
related:
  - "06 Execution/handoffs/durable-cost-evidence-v1-slice-prompt.md"
  - "06 Execution/reports/backed-depth-divergence-settlement-attempt-handoff-2026-07-06-v1.md"
---

# HANDOFF: durable cost evidence v1 slice prompt (docs-only) (2026-07-06)

## 1. objective

docs-only, no-spend: author the paste-ready implementation prompt for Durable Cost
Evidence v1 so the code slice can run safely after the 2026-07-07 settlement, with no
cohort in flight. this slice implements nothing.

## 2. outcome

COMPLETE. `06 Execution/handoffs/durable-cost-evidence-v1-slice-prompt.md` shipped,
grounded in read-only recon: metering ALREADY EXISTS and is pure
(model_metering.py: build_call_metadata with inputTokens/outputTokens/totalTokens/
estimatedTotalCost/pricingSource/costEstimateVersion, explicitly labelled
not-billing/stripe-remains-revenue-truth); the actual gap is transport + durability --
sports_analyzer._log_call_metadata emits the record to the devcore.cost LOGGER only,
so it never reaches the analyzer response, the AgentRun row, or /rows. the authored
prompt scopes the slice to: carry the existing record additively into the analyzer
response, persist it on AgentRun (single bounded json column preferred; precedent
PromptRouteProvenanceJson), expose trailing-optional camelCase fields on /rows
(precedent: the established additive pattern), /metrics byte-identical. supersededBy
rider: DEFERRED by default. mid-cohort execution: forbidden as a hard precondition.

## 3. repo state before / after

- dai: main @ `dbda7a8`, csproj phantom only. UNCHANGED (read-only in this slice).
- dai-vault before: `b8fdb00`, 5 ahead. after: +2 docs commits (slice prompt; this
  handoff), 7 ahead pre-push; push per instructions follows.

## 4. services used

none. devcore-sql + DevCore.Api remain running from the settlement window but were not
called. no statsapi, no agent-service, no docker changes.

## 5. work performed

read the queued-follow-up wording (pre-settlement-qa handoff s11) -> read-only recon of
model_metering.py, sports_analyzer.py metering call sites, AgentRun.cs entity fields,
AgentRunsController.cs, PromptRouteCalibrationExport.cs -> swept dai-vault for existing
cost/metering docs -> authored the paste-ready prompt with the 12 required content
blocks (header/mode flags, objective, non-goals, minimal field set reconciled to the
existing metering vocabulary, additive-only persistence strategy, generation-time
capture at the existing metering point, /rows exposure discipline, supersededBy
deferred, tdd test list, verification list, push instructions, 13-section handoff
requirement) -> this handoff -> commits -> push per instructions.

## 6. files changed

dai-vault only:
- `06 Execution/handoffs/durable-cost-evidence-v1-slice-prompt.md` (new)
- `06 Execution/reports/durable-cost-evidence-slice-prompt-handoff-2026-07-06-v1.md` (new)
dai: none.

## 7. db writes / side effects

0 db writes, 0 api calls, 0 migrations, no services started/stopped. the unreconciled
2026-07-06 cohort was not touched.

## 8. paid calls / cost

0 paid model calls, $0.00.

## 9. validation proof

- authored prompt contains the hard precondition "do not run mid-cohort" (verify no
  cohort awaiting settlement; STOP if any).
- supersededBy explicitly DEFERRED by default ("does NOT ride along unless the latest
  handoff at execution time explicitly orders it").
- additive-only constraints present for both persistence (additive migration only, no
  backfill without approval, null-safe historical runs) and read-model
  (trailing-optional /rows fields, /metrics byte-identical).
- push instructions present (fetch, status -sb, push only if main and not behind, no
  force push, stop-and-report on divergence, no unrelated files).
- dai untouched (git status: csproj phantom only, same as baseline).
- no paid calls; no services required.

## 10. what did not change

all dai code, schema, migrations (none created), prompts, routing, gates,
pooled_calibration.py, registry flags, buyer surfaces, the unreconciled cohort,
settlement state. gate 4 remains FALSE; gate 5 locked.

## 11. open issues

- the implementation slice is QUEUED, not scheduled: it must wait for the 07-07
  settlement AND a no-cohort-in-flight window.
- naming decision (json column vs scalar columns; modelName vs model) intentionally
  left to the implementation agent with the reconcile-to-existing-conventions rule.
- known untracked files (preflight manifest json, system-state synopsis) remain
  untracked by choice; not committed with this slice.

## 12. recommended next slice

Backed-Depth Divergence Settlement / Reconciliation v1 on 2026-07-07 morning (all 6
games Final), producing the first filled Gate-4 Evidence Readout. resume material:
blocked-settlement handoff sections 9/12/13.

## 13. suggested next prompt

use the blocked-settlement handoff section 13 with the gate-4 readout phase-4 addition:
verify all 6 statsapi states are Final before any settlement write; re-verify the
before column from a fresh /rows read; provenance source=statsapi with gamePk and final
score per the residue contract.
