# HANDOFF: Backed-Depth Cleanup + Divergence Capture Prep v1

**date:** 2026-07-05 · **mode:** measurement-integrity slice (NOT model tuning) · **end state:** COMPLETE (cleanup + report durability); divergence capture = plan only, paid capture NOT approved.

## 1. Objective
Three ordered objectives: (1) clean up the last dangling backed_depth registry row `822882` DET@TEX (run `c849433e`, prior no-lean canary); (2) make the completed 7/7 reconciliation report durable in git; (3) prepare (and only if `PAID_CAPTURE_APPROVED=true`, execute) the next measurement slice to find DAI-market divergence. `PAID_CAPTURE_APPROVED=true` was absent → default NO_SPEND_PREP.

## 2. Outcome
- `822882` **settled as no-decision (inconclusive)** via existing app path — real outcome recorded, no directional verdict fabricated. Backed_depth registry route now has **0 dangling rows**.
- Prior reconciliation report + a new cleanup report **committed** to dai-vault (not pushed).
- Divergence capture **plan** authored + committed. No paid capture (not approved) → no agent-service, no runs, no paid calls.

## 3. Repo State
### Before
- dai: `main` @ `32180df`, dirty (only pre-existing empty-diff `platform/dotnet/DevCore.Data/DevCore.Data.csproj` CRLF phantom), 0 ahead / 0 behind.
- dai-vault: `main` @ `5860e2b`, untracked = {`06 Execution/reports/reconciliation-last-cohort-2026-07-05-v1.md`, `06 Execution/system-state-synopsis-v1.md`}, 0 ahead / 0 behind.
### After
- dai: `main` @ `32180df` (UNCHANGED — no code edits), dirty (same csproj phantom only), 0 ahead / 0 behind.
- dai-vault: `main` @ `<tip = this handoff commit>`, **4 ahead / 0 behind** origin/main. Commits (oldest→newest): `e07744b` (reconciliation closeout), `35903ae` (dangling row cleanup), `85f1873` (divergence plan), `<this handoff>`. Still untracked (intentionally NOT committed): `06 Execution/system-state-synopsis-v1.md`. **Nothing pushed.**

## 4. Services Used
- devcore-sql (docker container): USED — read + the one no-decision write. Was already running from prior slice.
- DevCore.Api `:5007` (Development, dev-bypass tenant 1): USED — precheck, reconcile, calibration reads. Already running.
- MLB StatsAPI (`statsapi.mlb.com/api/v1.1/game/822882/feed/live`): USED — official final source, read-only.
- agent-service `:8000` (model path): NOT started. sports-app `:4201`: NOT started. → zero paid calls.

## 5. Work Performed
- Phase 0: verified both repos and services match handoff; StatsAPI reachable.
- Phase 1: DB-proved `822882` identity (mlb_statsapi, leanSide NULL, ExclusionReason NULL, hasOutcome 0); StatsAPI = Final (code F), TEX 0 / DET 3 → away_win; home/away refs match StatsAPI (no inversion); precheck = IdentitySafe; settled via `POST /reconcile` (away_win, 0-3) → SingleMatch, evalStatus=inconclusive.
- Phase 2: created cleanup report; committed prior reconciliation report + cleanup report as two docs commits; kept synopsis untracked.
- Phase 3 (NO_SPEND_PREP): authored + committed divergence capture plan; wrote this handoff.

## 6. Files Changed
- `dai-vault/06 Execution/reports/reconciliation-last-cohort-2026-07-05-v1.md` — prior 7/7 slice report, made durable (commit `e07744b`).
- `dai-vault/06 Execution/reports/backed-depth-dangling-row-cleanup-2026-07-05-v1.md` — NEW; documents 822882 no-decision settlement (commit `35903ae`).
- `dai-vault/06 Execution/reports/backed-depth-divergence-capture-plan-2026-07-05-v1.md` — NEW; NO_SPEND_PREP plan (commit `85f1873`).
- `dai-vault/06 Execution/reports/backed-depth-cleanup-divergence-prep-handoff-2026-07-05-v1.md` — NEW; this handoff.
- dai: none.

## 7. DB Writes / External Side Effects
- 1 `AgentRunOutcome` + 1 `AgentRunEvaluation` for run `c849433e` (822882): OutcomeStatus=away_win, HomeScore=0, AwayScore=3, EvalStatus=inconclusive, LeanSide=NULL, WinningSide=away; residue source=`statsapi_final`, sourceRef=`gamePk 822882`, notes non-thin.
- External: StatsAPI GET (read-only). No other side effects.

## 8. Paid Calls / Cost
- paid model calls: **0**
- estimated cost: **$0.00**
- proof: agent-service `:8000` never started; only StatsAPI (free) + devcore-sql + DevCore.Api reconcile/read endpoints called; AgentRuns count 273 unchanged (no generation).

## 9. Validation Proof
- 822882 settled inconclusive (not correct/incorrect): DB `EvalStatus=inconclusive`, `LeanSide=NULL` — no fabricated directional eval.
- No new agent runs: `SELECT COUNT(*) FROM AgentRuns` = **273** before and after.
- DAI hit rate unaffected: 822882 is no-decision; cohort read stays 5/7; route matchRate 0.682 unchanged; global noDecisionRows 17→18.
- 0 dangling backed_depth registry rows: route `unreconciledRows=0` (was 1).
- Report durable: dai-vault commits `e07744b`, `35903ae` present; synopsis remains untracked (not committed).
- No code/prompt/routing/confidence/buyer/migration/schema change: `dai` @ `32180df`, tree = csproj phantom only.
- Nothing pushed.

## 10. What Did Not Change
- prompts: unchanged
- routing: unchanged (registry stays default-off; `.env` untouched)
- confidence logic: unchanged
- buyer copy: unchanged
- migrations/schema: unchanged
- runtime behavior: unchanged (no dai code edits; only a data settlement write through the existing endpoint)

## 11. Open Issues
- Divergence measurement gap still open: the v8 backed-depth cohort had 7/7 DAI-market agreement → no independent-edge signal. Requires a divergence-seeking paid cohort (plan ready, approval-gated).
- `conclusionsAllowed` remains FALSE (confidence buckets thin: 0.80 n=3, 0.75 n=4 in-cohort).
- dai-vault has 4 unpushed commits; push only on explicit instruction.
- Local stack (devcore-sql + DevCore.Api :5007) left running.

## 12. Recommended Next Slice
Execute the divergence capture (PAID_CAPTURE) per `backed-depth-divergence-capture-plan-2026-07-05-v1.md`: on a game day, screen candidates via `/source-readiness`, apply the divergence prefilter (close favorites, |ΔimpliedProb| ≤ ~0.10), generate ≤12 registry-routed backed_depth runs, record every run, then settle in a separate slice once games are Final. Only with explicit `PAID_CAPTURE_APPROVED=true` + named gamePks + cost caps.

## 13. Suggested Next Prompt
```text
SLICE: Backed-Depth Divergence Capture (PAID) v1
Date: <YYYY-MM-DD>
PAID_CAPTURE_APPROVED=true
Caps: max 12 paid runs, model gpt-4o-mini only, 1 model call/run, total cost cap $0.05.

Execute dai-vault/06 Execution/reports/backed-depth-divergence-capture-plan-2026-07-05-v1.md.

Do (in order):
1. Verify repo state (dai clean except csproj phantom; dai-vault has the 4 unpushed docs commits from 2026-07-05).
2. Start devcore-sql + DevCore.Api :5007. Start agent-service :8000 ONLY now (registry canary env process-scoped, revert default-off after). Confirm cost logging active.
3. Screen game-day candidates via GET /api/agent-runs/source-readiness; keep only eligible starter_enriched_market_backed_depth games.
4. Apply the divergence prefilter (close market favorite, modest implied-prob gap, mixed book signals). RECORD the candidate slate BEFORE generating.
5. Generate <=12 runs via the existing app path (run-artifact-calibration.ps1 -Competition mlb). Do NOT retry a game for a different lean. Stop on any unexpected model path / promptSource != registry / cost cap / missing identity.
6. Record EVERY run (agreements + disagreements) with the Section 6 fields.
7. Do NOT reconcile now unless games are already Final + identity-safe (prefer separate settlement slice).
8. Write dai-vault/06 Execution/reports/backed-depth-divergence-capture-<date>-v1.md with paid-call count + cost + agreement/disagreement counts + identity safety + settlement readiness.

Hard constraints: no prompt/routing/confidence/buyer/schema/migration changes; registry stays default-off after; do not tune on results; do not push; no co-authored-by.
```

---
**Durable source of truth = this vault handoff.** Prior-slice detail: `reconciliation-last-cohort-2026-07-05-v1.md`; cleanup detail: `backed-depth-dangling-row-cleanup-2026-07-05-v1.md`; plan: `backed-depth-divergence-capture-plan-2026-07-05-v1.md`.
