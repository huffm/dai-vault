---
title: "Nightly Closeout 2026-07-09 -- reconciliation blocked (finals PARTIAL), no capture, WI-0002/WI-0003 registered, runtime stopped"
type: "evidence-report"
date: "2026-07-09"
status: "complete"
project: "DAI"
slice: "Nightly Closeout 2026-07-09"
repos:
  dai: "unchanged"
  dai-vault: "docs-only"
tags:
  - closeout
  - settlement
  - capture
  - work-items
related:
  - "06 Execution/reports/v2-accelerated-capture-day1-2026-07-09-v1.md"
  - "06 Execution/patterns/settlement-readiness-discipline-v1.md"
  - "02 Platform/system-development/work-items/WI-0002-artifact-chip-primitive-alignment.md"
  - "02 Platform/system-development/work-items/WI-0003-shared-chip-and-long-token-module-promotion.md"
---

# nightly closeout 2026-07-09

Facts, then interpretations. Zero operational writes tonight: 0 reconciliations, 0 captures,
0 paid model calls, 0 database mutations.

## repository truth at start

- dai: main @ `bb10c3c` = origin/main (WI-0001 integrated + pushed earlier tonight; branch
  deleted). Working tree: only the known stat-only `DevCore.Data.csproj` entry (index and
  worktree blobs identical, verified again this session). Untouched.
- dai-vault: main @ `0e5dbfc` = origin/main. Two pre-existing untracked files inspected:
  the 2026-07-06 preflight manifest is historical evidence of the already-settled 07-06
  cohort (settled 07-08) — remains intentionally untracked; the system-state synopsis
  follows standing precedent (left untracked across the 07-08 and 07-09 closeouts) —
  remains intentionally untracked. Neither staged nor modified.

## reconciliation (blocked, correctly)

- Candidates: the 8 day-1 v2 runs (only unreconciled cohort; 823281 was settled 07-08).
- Finals guard `check-settlement-finals.ps1` at 2026-07-10T01:58Z: **PARTIAL, exit 2 —
  5 final / 3 live** (823277 AZ@SD, 823034 MIL@STL, 822877 LAA@TEX in progress).
  Finals (for tomorrow's cross-check): 823359 ATL@PIT 10-5; 824816 CHC@BAL 2-3;
  823683 CLE@MIN 5-2; 824251 ATH@DET 1-4; 823846 SEA@MIA 4-8. Guard-reported, not yet
  independently re-verified at settlement time — re-verify before reconciling.
- Discipline: partial cohorts are never settlement-ready; the guard's verdict is the gate
  and bypassing it is out of bounds. The last live game (AZ@SD, 21:40 ET first pitch) could
  not finish before ~00:30 ET; the runbook already pairs this settlement with day-2 morning.
- **Preflighted 0, reconciled 0, rejected 3 (not final) + cohort rule (no partial
  settlement). Residue: all 8 day-1 runs remain eligible-pending finals.**

## capture (none, by rule -- not a failure)

- 07-09 capture is already complete at the cap (day-1: 8/8; two eligible games dropped by
  rank under the binding no-backfill directive). Remaining 07-09 games were in progress at
  assessment time — selection-before-outcomes forbids them; several are also the day-1
  cohort itself (duplicate-active rule).
- Day-2 (07-10) capture is runbook-ordered AFTER day-1 settlement ("capture without
  settlement is not evidence") inside the proven 10:00-13:00 ET window.
- Paid calls tonight: **0**. the-odds-api calls tonight: 0. statsapi: 1 free schedule read
  (the finals guard).

## calibration state (verified against persisted rows, read-only)

- /prompt-route-calibration/rows: 294 total, 255 active, **122 active settled**, 133 active
  unsettled; day-1's 8 rows present, active, 0 outcomes. **Duplicate-active gamePks: 0.**
- Market-opposed settled rows: **n=7** (persisted); binding interpretive numbers remain the
  07-08 n=7 checkpoint: 2/7 correct opposing, discrimination delta **-0.1184**
  (improved from -0.1321 purely by composition), edge-narrative NEGATIVE.
- Conclusions remain **blocked**: discrimination_inverted + insufficient_market_disagreement.
  Deliberate-disagreement ledger: 1 entry (823281, settled incorrect 07-08). No checkpoint
  fired tonight (no settlements). Sample remains insufficient; no defensible conclusion yet.
- Note: the closeout prompt's context (n=6, delta -0.1321, "next checkpoint n=7") predated
  the 07-08 checkpoint; persisted state and the 07-08/07-09 reports supersede it.

## work items registered (specs only, not implemented)

- **WI-0002 Artifact Chip Primitive Alignment** — BACKLOG, NOT AUTHORIZED. Disposition
  deliberately open (retain distinct / tokens only / shape modifier / full migration);
  inventory-before-mutation required; activation gate in spec.
- **WI-0003 Shared Chip and Long-Token Module Promotion** — BACKLOG, NOT AUTHORIZED.
  Activation gate: named, materially-compatible second consumer; no extraction for
  hypothetical reuse; "no extraction" is a valid outcome.
- MOC - DAI System Development updated with both entries. WI-0001 remains closed; its
  history untouched.

## runtime shutdown (verified)

- At task start (pre-existing): devcore-sql (up ~14h), DevCore.Api :5007 (PID 26904,
  running as compiled exe), agent-service :8000 (PID 26960 uvicorn reloader + elevated
  worker 5460 — restarted at some point after the day-1 closeout had left it stopped).
  Session-started: two orphaned sports-app vite servers on :4201 (PIDs 21360, 18480, from
  WI-0001 visual QA).
- Stopped tonight: both vite servers; agent-service reloader AND its inherited-socket
  worker (the worker survived the canonical stop script and taskkill — elevated; stopped
  via an unsandboxed Stop-Process); DevCore.Api (survived the CommandLine-matching stop
  script because it runs as `DevCore.Api.exe`, not `dotnet ...` — stop-platform-api.ps1
  match-pattern gap noted as a follow-up); devcore-sql via `docker stop` (graceful).
- Final verified state: **no listeners on 5007/8000/4200/4201/1433; 0 running containers;
  devcore-sql exited; no DevCore/python processes**. DAI runtime fully stopped.

## interpretations (bounded)

- Evidence unchanged tonight; the sample remains insufficient and discrimination remains
  inverted at n=7. Nothing tonight strengthens or weakens the edge narrative.
- The day-1 cohort is one finals-window away from settlement; tomorrow's pairing stands.

## follow-ups

1. 2026-07-10 ~10:20 ET: finals guard -> preflight-settlement -> identity /reconcile for
   the 8 day-1 runs (full residue), then n-recheck/readout, then day-2 capture per runbook.
2. `scripts/stop-platform-api.ps1`: extend process matching to catch `DevCore.Api.exe`
   (tonight it reported "no platform api process found" while the API listened on :5007).
3. Guard-classifier-conservatism note (822877) remains queued for the 07-11 baseline
   measurement, unchanged.

## next recommended action

Tomorrow's day-2 runbook execution, settlement first. Do not implement WI-0002/WI-0003;
their activation gates are not met.
