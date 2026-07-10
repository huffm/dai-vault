---
title: "V2 Day-1 Settlement and Day-2 Capture -- governed operational slice (2026-07-10)"
type: "plan"
date: "2026-07-10"
status: "complete"
project: "DAI"
slice: "V2 Day-1 Settlement and Day-2 Capture"
repos:
  dai: "unchanged"
  dai-vault: "docs-only"
tags:
  - settlement
  - capture
  - calibration
  - work-item
related:
  - "06 Execution/plans/v2-accelerated-capture-cadence-2026-07-09-v1.md"
  - "06 Execution/patterns/capture-closeout-run-eligibility-rule-v1.md"
  - "06 Execution/patterns/settlement-readiness-discipline-v1.md"
  - "06 Execution/patterns/cohort-selection-and-run-discipline-v1.md"
  - "06 Execution/reconciliations/canonical-reconciliation-residue-contract-v1.md"
  - "02 Platform/system-development/work-items/WI-0004-platform-api-shutdown-process-match.md"
---

# V2 day-1 settlement and day-2 capture -- governed operational slice

## identity

- **slice id:** OPS-2026-07-10-v1
- **title:** V2 Day-1 Settlement and Day-2 Capture
- **date:** 2026-07-10 (day 2 of the 2-day authorized cadence; day 3 = 07-11 wrap)
- **owner / execution role:** principal implementation agent, operator-authorized
- **status:** in-progress -> complete (front matter is the state holder)

## taxonomy decision (recorded on purpose)

This slice is **not** a `WI-####` work item, and that is a doctrine call, not an oversight.

The execution prompt proposed minting the next WI after WI-0003. Two canonical statements
forbid it:

- `02 Platform/system-development/operating-model.md` -> "what it is not": *"Not applicable
  to dai-vault strategy/calibration work, which keeps its own conventions."*
- `dai/.claude/skills/dai-system-development/SKILL.md` -> "When NOT to use": *"dai-vault
  strategy/calibration work (its own conventions)."*

The meaningful-change threshold requires a work item when a change alters **behavior, a
contract, UI, schema, or doctrine**. This slice alters none of those: it ingests settled
evidence, captures new prediction rows, and writes documentation. `_template.md` confirms
the taxonomy's shape is code work (test plan, branch, PR, affected surfaces, visual QA).

Forcing an operational cadence day into `WI-0004` would have corrupted a registry whose
three existing entries are all frontend implementation specs. Instead:

- **this slice** is governed by this OKF `plan` record, carrying the full governance field
  set (identity, scope, non-goals, authority, dependencies, risks, acceptance, verification,
  links, rollback, definition of done, lifecycle stages);
- **`WI-0004`** was minted for the platform-api shutdown-script defect discovered here,
  which genuinely *is* a code change and therefore belongs in that taxonomy.

The work-item lifecycle skill (`dai-system-development`) was used precisely as designed: it
routed the work **out** of the WI taxonomy and into the operational one.

## problem statement

The 2026-07-09 day-1 v2 cohort (8 runs) was captured cleanly but never settled: the
2026-07-09 nightly closeout hit `check-settlement-finals.ps1` PARTIAL (5 final / 3 live)
and correctly refused to settle a partial cohort, taking zero writes. Eight prediction rows
sat active and unreconciled, so the v2-era evidence they carry was not yet in the Gate 4
denominator, and day-2 capture was blocked behind the runbook's settlement-pairing rule.

## operational purpose

Convert 8 captured v2-era prediction rows into settled evidence; recompute Gate 4 from
persisted truth; then capture the day-2 cohort inside the authorized window so 07-11 can
wrap the cadence with two settled v2 cohorts.

## current evidence at intake (verified live, not assumed)

- dai `bb10c3c` = origin, 0/0; only the known stat-only `DevCore.Data.csproj` phantom
  (index blob == worktree blob == `285dd5e`; `git diff --numstat` empty).
- dai-vault `cbdf2c9` = origin, 0/0; two intentionally untracked files present.
- WI-0001 `complete`; WI-0002 / WI-0003 `blocked` (backlog, not authorized).
- Rows before settlement: 294 total / 255 active / 39 excluded / 125 outcomes /
  **0 duplicate-active gamePks**.
- Gate 4 before: `conclusionsAllowed=false`, failing
  `[discrimination_inverted, insufficient_market_disagreement]`; market-opposed n=7.

## scope

1. Cold-start only what settlement needs (docker, devcore-sql, DevCore.Api :5007).
2. Finals guard -> strict preflight -> identity `/reconcile` x8 with full residue.
3. Recompute Gate 4 from persisted rows; file the evidence readout.
4. Day-2 capture inside 10:00-13:00 ET, <=8 eligible runs, no backfill.
5. Post-capture verification, documentation, vault commit + push, runtime shutdown.

## non-goals

No prompt, model, temperature, confidence, decision, source-readiness, source-depth,
capture-ranking, market-agreement, reconciliation, or calibration-formula change. No Gate 4
threshold or conclusion-gating change. No schema, migration, buyer contract, tenant, or
billing change. No Tool Gateway enablement. No prompt-registry allowlist change. No
implementation of WI-0002, WI-0003, or WI-0004. No fix to `stop-platform-api.ps1`.

## execution authority

Operator-authorized under the 2-slate-day cadence
([[v2-accelerated-capture-cadence-2026-07-09-v1]]): <=8 eligible runs on 2026-07-10,
$0.05/day model cap, gpt-4o-mini, 10:00-13:00 ET window, no-backfill directive binding.
Authorization ends after the 07-11 wrap. Settlement is unpaid and not window-bound.

## dependencies

- Day-1 cohort games must be Final (external, calendar-gated).
- `devcore-sql` + `DevCore.Api` :5007 for all reads/writes; `agent-service` only at capture.
- the-odds-api quota for screening + generation.

## risks

- **Partial finals** -> settlement blocked (mitigated: finals guard is the hard gate).
- **Score drift** between readiness check and write (mitigated: scores re-verified from
  `feed/live` immediately before each write; a prior cohort had 3 games flip).
- **Duplicate-active regression** from capture (mitigated: sweep before, during, after).
- **Late capture** -> slate exhausted (mitigated: window compliance checked pre-spend;
  v1 failed exactly this way).
- **Deepening inversion** read as regression (mitigated: readout language rules; a gate
  failing honestly is not an operational failure).

## acceptance criteria

1. All 8 day-1 runs settled via identity `/reconcile`, each `SingleMatch`, each landing on
   its intended run id, or precisely blocked with recorded evidence.
2. Every settlement write carries complete residue (source, sourceRef, notes).
3. Exactly 8 rows change; 0 new runs; 0 duplicate outcomes; dup-active sweep 0 after.
4. Gate 4 recomputed from live `/rows` and quoted verbatim before interpretation.
5. Day-2 capture occurs only inside 10:00-13:00 ET, <=8 runs, eligible-only, no backfill;
   or is correctly declined by rule with the eligibility assessment recorded.
6. Spend <= $0.05; every paid call recorded.
7. WI-0002 / WI-0003 untouched; csproj phantom unstaged; the two untracked vault files
   untouched.
8. Runtime fully stopped and proven (ports + containers + processes).

## verification requirements

Persisted rows and the per-run `/evaluation` endpoint are authoritative. HTTP 200 is not
evidence. Before/after `/rows` snapshots are diffed to prove no unrelated run mutated. The
pooled report is run against live `/rows`, not a cached file.

## rollback / containment posture

Settlement writes are additive outcome+evaluation rows guarded by idempotency (409 on
re-write), direction integrity (422), and the residue contract (422). There is no destructive
path: a wrong settlement is contained by `/exclude`, never by deletion. Capture creates new
active runs only; the capture-closeout rule governs diagnostic exclusion. Zero migrations.

## lifecycle-stage tracking

| stage | state | evidence |
|---|---|---|
| 1 intake / truth | complete | repo + runtime truth recorded; discrepancy documented |
| 2 governance record | complete | this doc; WI-0004 minted for the code defect |
| 3 cold start | complete | SQL ready line; `/health 200`; pid recorded |
| 4 finals guard | complete | READY 8/8, exit 0 |
| 5 strict preflight | complete | exit 0, 8/8 ready, 0 warn, 0 block |
| 6 settlement | complete | 8x SingleMatch, residue complete, 6c/2i |
| 7 gate 4 readout | complete | recomputed live; `conclusionsAllowed=false` |
| 8 day-2 capture | complete | 8/8 registry v2, guard Pass 8, $0.00568, dup-active 0 |
| 9 documentation | complete | settlement + readout + capture report + manifest + WI-0004/0005 |
| 10 commit / push | complete | vault only; dai untouched at `bb10c3c` |
| 11 shutdown | complete | ports 5007/8000/4200/4201/1433 free; devcore-sql stopped |

## deviations from the execution prompt (recorded, with evidence)

1. **Runtime was not cold at intake.** The prompt asserted Docker Desktop down, `devcore-sql`
   stopped, 1433 free. Actual: Docker Desktop up since 08:32:58 ET, `devcore-sql` up since
   08:33:12 ET (`restartPolicy=no`, so not policy-driven), 1433 held by Docker's port proxy
   (`wslrelay` + `com.docker.backend`), no Windows MSSQL service. Same canonical image, single
   container. Benign; SQL readiness was verified from the container log
   (`SQL Server is now ready for client connections`, 12:33:19Z) rather than from "Up" status.

2. **No `WI-####` was minted for this slice.** See the taxonomy decision above. `WI-0004` and
   `WI-0005` were minted instead for the two code defects discovered.

3. **Day-2 capture required an operator decision mid-slice.** The permission classifier blocked
   `DAI_MLB_REGISTRY_PROMPT_CANARY=1`, reading it as a forbidden prompt-registry flag flip. The
   operator then explicitly authorized it, process-scoped, for this capture only. Without that
   authorization the capture would have run on the live path with `attributionStatus=partial`.

4. **The first screen was discarded.** A poisoned starter cache false-negated 6 of 15 candidates
   (WI-0005). The paced 15-of-15 re-screen after an API restart is the authoritative screen. No
   spend occurred against the bad screen.

5. **`DevCore.Api` was restarted mid-slice** (after settlement, before capture) solely to clear
   the in-process `IMemoryCache`. No code change; settlement writes were already durable in SQL.

## acceptance criteria result

1. 8/8 settled, all SingleMatch, all on intended runs — **met**
2. residue complete on all 8 — **met**
3. exactly 8 rows changed, 0 new runs, 0 dup outcomes, dup-active 0 — **met**
4. Gate 4 recomputed live and quoted verbatim before interpretation — **met**
5. capture in-window, 8 runs, eligible-only, no backfill — **met** (15/15 eligible; cap bound)
6. spend $0.00568 <= $0.05; every paid call recorded — **met**
7. WI-0002/0003 untouched; csproj unstaged; 2 untracked vault files untouched — **met**
8. runtime fully stopped and proven — **met**

## required links

- cadence runbook: [[v2-accelerated-capture-cadence-2026-07-09-v1]]
- day-1 capture report: `06 Execution/reports/v2-accelerated-capture-day1-2026-07-09-v1.md`
- prior closeout: `06 Execution/reports/nightly-closeout-2026-07-09-v1.md`
- settlement report: `06 Execution/reconciliations/v2-day1-cohort-settlement-2026-07-10-v1.md`
- gate-4 readout: `06 Execution/reports/gate4-evidence-readout-v2-day1-2026-07-10-v1.md`
- day-2 capture report: `06 Execution/reports/v2-accelerated-capture-day2-2026-07-10-v1.md`
- day-2 frozen slate + capture manifest: `06 Execution/reports/v2-capture-day2-frozen-slate-manifest-2026-07-10-v1.json`
- follow-up work item: [[WI-0004-platform-api-shutdown-process-match]]
- follow-up work item: [[WI-0005-starter-retrieval-caches-transport-failures]]
- handoff: `06 Execution/handoffs/v2-day1-settlement-day2-capture-handoff-2026-07-10-v1.md`
- slice log: `06 Execution/handoffs/current-slice.md` (appended)

## definition of done

All eight acceptance criteria met or precisely blocked with recorded evidence; Gate 4
recomputed; documentation matches persisted state; vault commit pushed; runtime stopped;
07-11 starting point recorded; WI-0002/0003/0004 all remain unauthorized and unimplemented.
