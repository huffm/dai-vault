---
title: "Pre-Settlement QA Script + Cohort Selection Discipline 2026-07-06 v1"
type: "report"
date: "2026-07-06"
status: "COMPLETE -- script shipped + validated against the live 6-run cohort; discipline pattern + slate template added"
project: "DAI"
slice: "Pre-Settlement QA Script + Cohort Selection Discipline v1"
repos:
  dai: "script-only (scripts/dev/sports/preflight-settlement.ps1)"
  dai-vault: "docs-only"
tags:
  - calibration
  - cohort
  - qa
  - tooling
  - pattern
related:
  - "06 Execution/reports/backed-depth-divergence-cohort-integrity-qa-2026-07-06-v1.md"
  - "06 Execution/reports/interim-system-improvement-clinical-audit-2026-07-06-v1.md"
  - "06 Execution/patterns/cohort-selection-and-run-discipline-v1.md"
  - "06 Execution/patterns/frozen-cohort-slate-template-v1.md"
---

# Pre-Settlement QA Script + Cohort Selection Discipline 2026-07-06 v1

## 1. objective

Convert the manual pre-settlement QA (Cohort Integrity QA v1) into a reusable read-only
script, and codify cohort selection / capture-run discipline so future paid captures are
measurement-grade by construction. No-spend; no settlement; no model behavior change.

## 2. repo state

- Before: dai `d79c38f` (csproj phantom only, 0/0); dai-vault `ab79a1a` (4 ahead, untracked
  synopsis). Settlement-override gate checked: at 16:55 ET all 6 cohort games were still
  Pre-Game/Scheduled -> no finals available -> tooling slice proceeds.
- After: dai = `d79c38f` + 1 script commit; dai-vault = `ab79a1a` + 1 docs commit.

## 3. manual QA checklist converted

From QA v1: cohort membership (exactly one active run per gamePk, completed, not excluded),
identity (sourceProvider + externalGameId), registry provenance (promptSource, fallback,
regime, attribution), decision fields (lean, confidence, evidenceRichness), market baseline
(consensus side, implied probs, book count, derived agreement), settlement state (0
outcomes/evaluations), and per-game `/reconcile-precheck` (SingleMatch). All automated
except: SupersededBy (not exposed by read endpoints; covered indirectly by active-run
uniqueness + precheck) and per-run cost evidence (stdout-only; stays manual/report-level).
Both are recorded in the manifest's `limitations`.

## 4. script added

`dai/scripts/dev/sports/preflight-settlement.ps1` (PS7, same conventions as
run-artifact-calibration.ps1: param block, bearer/dev-bypass auth, colored output).

Read-only by construction: the only endpoints called are GET
`/api/agent-runs/prompt-route-calibration/rows` (primary read model -- one call for the
whole cohort), GET `/api/agent-runs/{id}` (status), GET `/api/agent-runs/{id}/evaluation`
(404 = absent), GET `/api/agent-runs/reconcile-precheck`. It never calls `/reconcile`,
never creates runs, never writes DB rows, never touches agent-service.

## 5. script usage

```powershell
.\scripts\dev\sports\preflight-settlement.ps1 `
  -Competition mlb `
  -GamePks 822958,822712,824900,823036,823282,823205 `
  -ExpectedRunPrefixes ac31433e,ad31433e,b331433e,b431433e,b631433e,b731433e `
  -RequireRegistry -RequireBackedDepth -RequireUnreconciled `
  -OutputPath "..\dai-vault\06 Execution\reports\preflight-settlement-manifest-2026-07-06-v1.json"
```

Inputs: `-Competition` (required), `-GamePks` or `-ManifestPath`; optional `-OutputPath`,
`-ApiBaseUrl` (default :5007), `-SourceProvider` (default mlb_statsapi),
`-ExpectedRunPrefixes`, `-RequireRegistry` / `-RequireBackedDepth` / `-RequireUnreconciled`
(escalate those check groups from warnings to blockers), `-FailOnWarnings`, `-JsonOnly`,
`-BearerToken`. Exit codes: 0 ready / 1 warnings with -FailOnWarnings / 2 blockers /
3 script-config-api failure.

## 6. manifest output

JSON manifest (summary + per-run records + blockers/warnings/limitations) plus a
human-readable table. Per-run record includes gamePk, agentRunId/prefix, matchup, status,
lean/confidence/evidenceRichness, marketFavorite/bookCount/impliedProbGap/marketAgreement
(from the persisted MarketSnapshotBatch fields on `/rows` -- structured, decision-time),
provenance (promptSource/regime/fallback/attribution), precheckStatus, outcome/evaluation
presence, readyForSettlement.

Raw JSON manifests are regenerable output and are left uncommitted (this report embeds the
validated summary).

## 7. cohort selection discipline added

`06 Execution/patterns/cohort-selection-and-run-discipline-v1.md` -- 11 selection
principles (objective-first, freeze-before-spend, decision-time-data-only, immutable slate,
no mid-capture tuning, capture!=settlement), a 10-dimension candidate scoring model with
Primary/Secondary/Exclude/Blocker labels, a 13-step capture run discipline (ending with
"run preflight-settlement.ps1 before the settlement slice"), the 7 required cohort
artifacts, and 7 anti-patterns (incl. spending-because-time-is-available and
backfilling non-decision-time market data).

## 8. frozen slate template added

`06 Execution/patterns/frozen-cohort-slate-template-v1.md` -- copy-paste markdown skeleton
mirroring the capture-v2 slate that passed QA (freeze statement, objective, caps, window,
universe, screening timestamps/method, classification table, selections/exclusions with
reasons, stop conditions, pre-generation baselines, post-capture generation results,
settlement plan). Placed in `patterns/` (no `templates/` folder exists in the vault).

## 9. validation against the 6-run cohort (live)

Strict mode (all three Require switches + expected prefixes):

```text
summary: target 6 | found 6 | ready 6 | warnings 0 | blockers 0 | agree 5 | disagree 1
exit code: 0
```

- Divergence run correctly identified: 823036 milwaukee-brewers @ st-louis-cardinals,
  lean home vs marketFavorite away, marketAgreement=false, impliedProbGap 0.037 (persisted
  median de-vig), SingleMatch, readyForSettlement=true.
- All 6: promptSource=registry, regime backed_depth, attribution complete, precheck
  SingleMatch, outcome/evaluation absent.
- Negative path: unknown gamePk -> blocker "no active run found" -> exit 2. ✓
- Manifest written to `06 Execution/reports/preflight-settlement-manifest-2026-07-06-v1.json`
  (left uncommitted by convention choice).

Two implementation defects were found and fixed during validation (both PowerShell
semantics, not data issues): Invoke-RestMethod returning the rows array as a single
non-enumerated item (fixed by pipeline flattening) and `$run?.prop` parsing as a variable
named `run?` (fixed with `${run}?.prop`).

## 10. limitations

- SupersededByAgentRunId not exposed by any read endpoint -> not checked directly (covered
  indirectly; a future read-model field could close this).
- Per-run cost evidence remains stdout-only (Durable Cost Evidence v1 is the follow-up).
- Source-depth groups verified via selectedDataRegime, not per-artifact sourceDepth detail
  (one /rows read instead of N artifact fetches; the regime encodes both groups).
- `/rows` is tenant-scoped and unpaginated (279 rows today) -- fine at current scale.

## 11. what did not change

Runtime app code, prompts, registry recipes, routing, confidence generation, calibration
gate, buyer copy, schema/migrations, captured artifacts: unchanged. agent-service never
started. 0 paid calls. 0 DB writes (AgentRuns 279, outcomes 112, evaluations 112 before and
after all script executions). The only dai change is the new dev script.

## 12. recommended next slice

**Backed-Depth Divergence Settlement / Reconciliation v1** once official finals for
2026-07-06 exist (cohort games end ~00:30 ET): run `preflight-settlement.ps1` (strict) as
Phase 0, then settle all 6 via identity `/reconcile`, write outcomes/evaluations, re-read
pooled calibration. Watch 823036.
