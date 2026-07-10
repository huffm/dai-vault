---
title: "Handoff -- V2 day-1 settlement + day-2 capture (2026-07-10)"
type: "handoff"
date: "2026-07-10"
status: "complete"
project: "DAI"
slice: "V2 Day-1 Settlement and Day-2 Capture"
repos:
  dai: "unchanged"
  dai-vault: "docs-only"
tags:
  - handoff
  - settlement
  - capture
related:
  - "06 Execution/plans/v2-day1-settlement-day2-capture-slice-2026-07-10-v1.md"
  - "06 Execution/reconciliations/v2-day1-cohort-settlement-2026-07-10-v1.md"
  - "06 Execution/reports/gate4-evidence-readout-v2-day1-2026-07-10-v1.md"
  - "06 Execution/reports/v2-accelerated-capture-day2-2026-07-10-v1.md"
  - "02 Platform/system-development/work-items/WI-0004-platform-api-shutdown-process-match.md"
  - "02 Platform/system-development/work-items/WI-0005-starter-retrieval-caches-transport-failures.md"
---

# handoff -- v2 day-1 settlement + day-2 capture

## 1. objective

Settle the 2026-07-09 v2 day-1 cohort (blocked the prior night at a PARTIAL finals gate),
recompute Gate 4, then capture the day-2 cohort inside the authorized window. Day 2 of a
2-slate-day cadence whose authorization ends with the 07-11 wrap.

## 2. outcome

**SLICE COMPLETE.** 8/8 settled (6c/2i), Gate 4 recomputed (`conclusionsAllowed=false`),
8/8 captured registry-routed v2 (guard Pass 8/8, $0.00568), 0 hard stops, runtime stopped.
Two code defects found and registered as work items; neither fixed here.

## 3. repo before

- dai `bb10c3c` == origin/main, 0/0. Only ` M DevCore.Data.csproj` (stat-only phantom:
  index blob == worktree blob == `285dd5e`, `git diff --numstat` empty).
- dai-vault `cbdf2c9` == origin/main, 0/0. Two intentionally untracked files present.
- WI-0001 `complete`; WI-0002 / WI-0003 `blocked`.

## 4. repo after

- dai **unchanged** `bb10c3c`, 0/0, csproj phantom still unstaged. No code commit.
- dai-vault: one docs commit, pushed. Two untracked files still untracked.

## 5. services

- Docker Desktop + `devcore-sql` were **already up** at intake (08:32:58 / 08:33:12 ET), contrary
  to the prompt's "cold" assumption; `restartPolicy=no`, so externally warmed. 1433's two
  listeners are Docker's proxy (`wslrelay`, `com.docker.backend`), not a rogue SQL.
- SQL readiness proven from the container log, not from "Up".
- `DevCore.Api` :5007 started (pid 27608), **restarted mid-slice** (pid 12404) to clear the
  poisoned `IMemoryCache`, `/health 200` both times.
- `agent-service` :8000 started **only for capture** (pid 29496) with
  `DAI_MLB_REGISTRY_PROMPT_CANARY=1` in the child process env; stopped after. `.env` never written.
- sports-app never started.
- All stopped at close.

## 6. work performed

1. Skills gate; work-item lifecycle consulted (`dai-system-development`) and it routed this
   slice **out** of the WI taxonomy.
2. Governance records: operational slice record; WI-0004; WI-0005; MOC updated.
3. Cold-start / readiness verification.
4. Finals guard READY 8/8 → strict preflight exit 0 → per-game independent re-verification →
   8 identity `/reconcile` with full residue → per-run read-back of run + evaluation.
5. Gate 4 recomputed from live `/rows`.
6. Day-2 screen (defect found, cache cleared, paced re-screen), slate frozen pre-spend,
   canary-first capture of 8.
7. Documentation, commit, push, shutdown.

## 7. files

Created: slice record; settlement reconciliation; gate-4 readout; day-2 capture report;
frozen-slate manifest (JSON); WI-0004; WI-0005.
Updated: `MOC - DAI System Development.md`; `current-slice.md` (appended, 13274 → 13347 lines).
Untouched: WI-0002, WI-0003, `DevCore.Data.csproj`, `preflight-settlement-manifest-2026-07-06-v1.json`,
`system-state-synopsis-v1.md`.

## 8. db side effects

- 8 outcome + 8 evaluation rows (day-1 settlement). outcomes 125 → 133.
- 8 new `AgentRun` rows (day-2 capture). rows 294 → 302.
- 0 exclusions, 0 migrations, 0 schema change, 0 duplicate-active at every sweep.
- 0 diagnostic runs created ⇒ no capture-closeout exclusion owed.

## 9. paid calls + cost

- day-1 settlement: **0** paid calls (agent-service never started for it).
- day-2 capture: **8** gpt-4o-mini calls, one `create()` each, est **$0.00568** of the $0.05 cap.
- the-odds-api: ~44 calls (15 poisoned screens + 6 retries + 6 re-screens + 15 paced re-screens
  + 2 h2h reads + 8 generations). The cache defect roughly doubled screen cost — attributable to
  WI-0005, not to the ranking method.
- Cost figures are metering estimates, not billing truth (durable per-run sink still missing).

## 10. validation

- Settlement: HTTP 200 never trusted; each write confirmed by reading back the run and
  `GET /{id}/evaluation`, and by diffing full `/rows` before/after (exactly 8 rows changed;
  changed gamePk set == target set; 0 new runs).
- Capture: every run verified on `/rows` before the next was generated (registry / v2 / recipe /
  regime / attribution complete / no fallback), plus a dup-active sweep after each.
- Gate 4 quoted verbatim from the canonical pooled report before any interpretation.

## 11. what did NOT change

Prompt text, model, temperature/token limits, confidence, decision behavior, source-readiness
semantics, source-depth rules, capture ranking, market-agreement derivation, reconciliation
semantics, calibration formulas, Gate 4 thresholds, conclusion gating, schema, migrations, buyer
contracts, tenant/billing, Tool Gateway, registry allowlist. Registry routing remains
**default-off**. The registry path was used only after proving byte-identity with the live prompt
(`attributionReason` on all 8 runs), so the model saw identical bytes either way.

## 12. open issues

- **WI-0004** (BACKLOG, NOT AUTHORIZED): `stop-platform-api.ps1` matches only the `dotnet.exe`
  host; `DevCore.Api.exe` keeps 5007 bound while the script exits 0. Reproduced live. Also a
  latent `$matches` automatic-variable collision. Not fixed; shutdown here bypassed the script.
- **WI-0005** (BACKLOG, NOT AUTHORIZED): `MlbStarterClient` caches transport failures for 30 min
  as "no starters announced". Cost 6 false-negative candidates of 15. Not fixed; contained by an
  API restart + paced serial re-screen.
- `/rows` field-name traps that cost time and nearly caused false conclusions:
  `matchedOutcome` means *an outcome row exists*, not *the lean was right*
  (`PromptRouteCalibrationExport.cs:204`); the recipe field is `selectedPromptRecipeId`, not
  `recipeId`; there is no `evalStatus` on `/rows`.
- Discrimination inversion deepened to −0.1343 and is **not volume-purchasable**; only correct
  `gte_0.80` rows can move it. Day 2 contributed exactly one 0.80 row.
- Two cadence days produced 1 market-opposed row total (both days combined), against a sub-gate
  needing +3 settled opposed rows.

## 13. next slice (2026-07-11 wrap — authorization ENDS here)

Smallest precise sequence:

1. **Cold start:** Docker → `docker start devcore-sql` → wait for
   `SQL Server is now ready for client connections` in `docker logs` → `scripts/start-platform-api.ps1`
   → `/health 200` on :5007. Leave agent-service DOWN (no model path needed; settlement is free).
2. **Finals guard** (do not settle a partial cohort):
   `check-settlement-finals.ps1 -Competition mlb -GamePks @(824493,822955,823278,823845,824252,823357,823685,823604) -CheckLocalRows -RequireUnreconciled -FailOnPartial`
   Expect READY 8/8, exit 0. First check after ~01:00 ET (last first-pitch 19:15 ET).
   **`-GamePks` must be a real array**, not a comma-joined string.
3. **Strict preflight:**
   `preflight-settlement.ps1 -Competition mlb -GamePks @(...) -RequireRegistry -RequireBackedDepth -RequireUnreconciled -FailOnWarnings`
   Expect exit 0, 8 ready, 0 warnings, 0 blockers, agree 7 / disagree 1.
4. **Settle:** re-verify each final score + home/away identity from statsapi `feed/live`
   immediately before its write, then identity `POST /api/agent-runs/reconcile` ×8 with full
   residue (`source=statsapi`, `sourceRef="gamePk <pk> final away <a> home <h>"`,
   notes naming the 2026-07-10 v2 day-2 cohort + verification). Read back run + `/evaluation`.
5. **Final gate-4 readout:** rerun `scripts/pooled_calibration_report.py --url .../rows`; quote
   `conclusionsAllowed`, `failingReasons`, both populated regions, the discrimination delta,
   `marketDisagreementN`, `marketCoverage` verbatim before interpreting. Expect the ledger to move
   n=7 → 8 (still < 10 ⇒ still unreadable) and `gte_0.80` to gain one row (n=17 → 18).
6. **823845 CLE@MIA is the row to watch.** It is the first captured `DeliberateDivergence`
   (`prose_acknowledges_market_opposition`). Settling it creates the **second**
   `CountsAsCandidateEdge` ledger entry (after 823281, currently 0/1). Candidate-edge language is
   permitted only for deliberate rows, and **not as an edge claim at n=2** — report the ledger,
   do not narrate an edge.
7. **Hardened-Regime Baseline Measurement v1:** compare v2-era guard outcomes (16 captured rows
   across both days: Pass 15 / Unclear 1 / FAIL 0) against the **frozen v1 baseline
   Pass 72 / FAIL 10 / Unclear 203 (285 rows)**. Standing rule: **never pool v1 and v2 attribution
   rates.** Direction only at this n. Corpus FAIL has held at 10 throughout.
8. **822877 classifier note (carry verbatim):** its `UnclearMarketAttribution` /
   `both_market_directions_asserted` is confirmed **classifier ambiguity** (opponent-as-object: one
   market clause names both teams, so `DetectClaimedSides` returns `{home,away}`), **not** a model
   contradiction and **not** the FAIL hard stop. Deferred fix, not a defect in the run.
9. **Cadence closeout:** wrap report; capture authorization **expires**. Any further capture needs
   a fresh operator go. Do not implement WI-0002 / WI-0003 / WI-0004 / WI-0005.
10. **Shutdown:** stop the :5007 listener **and** its `dotnet.exe` host by pid — do **not** trust
    `stop-platform-api.ps1` (WI-0004). Then `docker stop devcore-sql`. Verify 5007/8000/4200/4201/1433
    all free and 0 containers.

## 14. skills used

`dai-skill-router` (gate), `dai-slice-runner` (lifecycle), `dai-system-development` (work-item
lifecycle authority — it routed this slice out of the WI taxonomy), `dai-grill-with-vault`
(reconciled the prompt's claims against persisted truth; found the runtime and taxonomy
discrepancies), `dai-test-discipline` (no code change ⇒ no suite run), `dai-docs-architect` (OKF
placement + front matter), `dai-token-tight`, `dai-agent-handoff` (this doc),
`superpowers:verification-before-completion` (evidence before every claim).

**Named in the prompt but not loadable as skills:** "cohort-selection discipline",
"pre-settlement QA discipline" (both exist as **vault patterns**, used as such), and a
"work-item lifecycle skill" (resolves to `dai-system-development`, a router). Fallback: the
canonical vault patterns and the `check-settlement-finals.ps1` / `preflight-settlement.ps1`
scripts, which are the encoded discipline.
