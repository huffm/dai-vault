---
title: "HANDOFF: 2026-07-07 Capture Cohort Settlement -- COMPLETE (6/6 settled, 4 correct / 2 incorrect; coverage sub-gate now MET)"
type: "handoff"
date: "2026-07-08"
status: "COMPLETE -- 6 reconcile writes, second filled gate-4 readout shipped"
project: "DAI"
slice: "RESUME: 2026-07-07 Capture Cohort Settlement v1"
related:
  - "06 Execution/reports/backed-depth-capture-cohort-2026-07-07-v1.md"
  - "06 Execution/reports/backed-depth-capture-settlement-attempt-handoff-2026-07-07-v2.md"
  - "06 Execution/reports/gate4-evidence-readout-backed-depth-capture-2026-07-07-v1.md"
  - "06 Execution/reports/market-attribution-fidelity-guard-handoff-2026-07-07-v1.md"
---

# HANDOFF: 2026-07-07 capture cohort settlement -- COMPLETE

## 1. objective

settle the 2026-07-07 backed_depth capture cohort (6 runs) through identity /reconcile
after authoritative finals, produce the filled gate-4 evidence readout with the 824820
accidental-divergence annotation and live fidelity-guard fields, and check the
marketDisagreementN=7 re-projection checkpoint.

## 2. outcome

COMPLETE. finals gate READY 6/6 (third attempt; the two 07-07 blocks were timing-only,
as diagnosed). all 6 settled via identity POST /reconcile, SingleMatch on every write:
**4 correct / 2 incorrect (0.6667)**. the market-opposed row 824820 CHC@BAL settled
INCORRECT (dai home Orioles 0.75 vs staged market away Cubs; final CHC 5 - BAL 2) --
live /rows guard fields exactly as predicted: FailMarketAttributionMismatch /
prose_claims_home_but_staged_consensus_is_away / AccidentalDivergence. gate-4 movement:
**market coverage crossed its threshold (0.5918 -> 0.6154 vs 0.60) and
insufficient_market_coverage dropped from failingReasons** (now 2: discrimination_inverted,
insufficient_market_disagreement). discrimination inversion deepened (delta -0.0882 ->
-0.1321; the lone 0.80-conf run 822713 settled incorrect). marketDisagreementN=6 --
one short of the n=7 re-projection checkpoint (not triggered). conclusionsAllowed false.
settled market-opposed ledger: n=6, 2c/4i, all accidental; deliberate candidate-edge
ledger still EMPTY.

## 3. repo state

### before
- `dai`: main @ `a0db824`, 0/0, phantom DevCore.Data.csproj delta only.
- `dai-vault`: main @ `de2e3f0`, 0/0, known untracked noise
  (preflight-settlement-manifest-2026-07-06-v1.json, system-state-synopsis-v1.md).

### after
- `dai`: UNCHANGED (main @ `a0db824`, phantom only).
- `dai-vault`: readout + this handoff + current-slice.md append committed and pushed
  (sha in the rolling log entry); untracked noise untouched.

## 4. services used

- Docker Desktop: was DOWN at session start; started this session.
- devcore-sql: started (`docker start devcore-sql`); left running.
- DevCore.Api :5007: started via `dotnet run --launch-profile http` (background);
  left running. /rows read x3 (pre-flight probe, before snapshot, after snapshot).
- agent-service: NOT started; zero model calls.
- statsapi: one free schedule read (check-settlement-finals.ps1).
- the-odds-api: untouched.

## 5. work performed

skills gate -> baselines (dai `a0db824`, vault `de2e3f0`) -> phase 1 finals gate:
check-settlement-finals.ps1 READY 6/6 Final/F, exit 0 -> stack brought up (docker,
devcore-sql, DevCore.Api) -> strict preflight-settlement.ps1 with capture report sec 10
args: exit 0, 6/6 ready, SingleMatch, prefixes verified, 0 warnings 0 blockers ->
fresh before snapshot (/rows: 285 total / 118 settled; 6/6 cohort rows active
unreconciled; 824820 guard fields confirmed pre-write) -> identity POST /reconcile x6
with full residue provenance -> post-verification (/rows: 124 settled; residue non-null
6/6; matchedOutcome set 6/6) -> pooled_calibration_report.py on before+after snapshots
-> filled readout (gate4-evidence-readout-backed-depth-capture-2026-07-07-v1.md, taxonomy
language rules applied) -> this handoff -> rolling log append -> commit + push.

## 6. files changed

- dai: none.
- dai-vault: `06 Execution/reports/gate4-evidence-readout-backed-depth-capture-2026-07-07-v1.md`
  (new), this handoff (new), `06 Execution/handoffs/current-slice.md` (append).

## 7. db writes / external side effects

- 6 AgentRunOutcome + 6 AgentRunEvaluation rows staged/written via the 6 identity
  /reconcile calls (the shared AddOutcomeAndEvaluation path). rows 285 unchanged;
  settled 118 -> 124.
- no other writes: no runs created, no exclusions, no market snapshots, no flags.

## 8. paid calls / cost

- paid model calls: 0. estimated cost: $0.00.
- proof: agent-service never started; the settlement path is db + statsapi only.

## 9. validation proof

- finals: check-settlement-finals.ps1 exit 0, 6/6 abstractGameState=Final
  codedGameState=F with final scores (1-3, 5-2, 4-6, 6-3, 1-4, 8-1).
- preflight: exit 0, found 6 / ready 6 / warnings 0 / blockers 0 / agree 5 / disagree 1;
  manifest in session scratchpad. NOTE: pass -GamePks as a REAL ARRAY when invoking
  in-process with `&` -- a single comma-joined string is not split by this script
  (first invocation failed exit 2 with "no active run found"; zero risk, read-only).
- reconcile responses: 6x SingleMatch, evaluated run prefixes
  9e2c433e/a32c433e/a92c433e/aa2c433e/ac2c433e/b32c433e, evals
  correct/incorrect/correct/incorrect/correct/correct.
- post /rows: settled 124 (+6); settlementSource/sourceRef/notes non-null on all 6;
  824820 fidelity fields quoted verbatim in the readout.
- summary: pooled_calibration_summary before (116/98/n=5/0.5918/-0.0882/3 failing) vs
  after (122/104/n=6/0.6154/-0.1321/2 failing) -- deltas internally consistent with +6
  directional rows and the 0.80 miss.

## 10. what did not change

- prompts: unchanged. routing: unchanged (registry default-off; canary untouched).
- confidence logic: unchanged. buyer copy: unchanged.
- migrations/schema: unchanged. runtime behavior: unchanged (no dai code edits).
- gate-4 criterion/thresholds: unchanged -- the coverage sub-gate flipping to met is
  data movement, not a threshold edit.
- capture cadence: still PAUSED; resumption needs operator re-approval per
  market-attribution-fidelity-guard-v1.md sec 10.

## 11. open issues

- marketDisagreementN=6: the n=7 re-projection checkpoint fires on the NEXT settled
  market-opposed row.
- Run Identity Hygiene v1 pending: 824662 (2cde423e unsettled + 4cbd433e settled) and
  823281 (6a37433e + 1ede423e, both active/unsettled) duplicate-active pairs must be
  resolved before anyone settles those gamePks.
- Prompt Market Context Hardening v1 pending (approval-gated, paid canary; guard
  baseline Pass 72 / FAIL 10 / Unclear 203 on 285 rows).
- durable per-run cost sink still missing (unchanged).
- services left RUNNING (docker/devcore-sql/DevCore.Api) though found down at session
  start -- stop them if the machine should return to its found state.

## 12. recommended next slice

**Run Identity Hygiene v1** (per the fidelity-guard handoff sequence: settlement done ->
hygiene -> prompt hardening): decide supersession/exclusion for the 824662 and 823281
duplicate-active pairs via POST /{id}/exclude with documented reasons; read-only audit
first; no settlement of either gamePk until resolved.

## 13. suggested next prompt

```text
Run Identity Hygiene v1 (dai + dai-vault). Context: 2026-07-07 cohort settled 6/6
(see 06 Execution/reports/backed-depth-capture-settlement-handoff-2026-07-08-v1.md).
Two gamePks have duplicate ACTIVE rows: 824662 (2cde423e 06-28 unsettled + 4cbd433e
06-29 settled) and 823281 (6a37433e + 1ede423e, both 06-28, both unsettled, both
classified DeliberateDivergence by the fidelity guard). Read-only audit first: pull
/rows + artifacts for all four runs, establish which row per gamePk is authoritative
(freshness, provenance, attribution status), and propose supersession/exclusion per
the Run Eligibility and Supersession Contract v1. Apply POST /{id}/exclude only after
presenting the proposal for approval in-session. Constraints: no model calls, no
settlement writes, no prompt/gate/threshold changes; if 823281 exclusion decisions
would create the first deliberate ledger entries, flag that explicitly and stop for
approval. Close with a continuation-grade handoff + current-slice.md append.
```
