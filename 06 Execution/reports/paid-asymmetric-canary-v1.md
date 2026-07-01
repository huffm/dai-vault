---
title: "Paid Asymmetric Canary v1"
type: "evidence-report"
date: "2026-06-30"
status: "blocked"
project: "DAI"
slice: "Paid Asymmetric Canary v1"
repos:
  dai: "unchanged"
  dai-vault: "docs-only"
tags:
  - prompt-registry
  - asymmetric
related:
  - "06 Execution/reports/asymmetric-registry-canary-v1.md"
---

# Paid Asymmetric Canary v1 -- Evidence Report

**status:** BLOCKED on candidate availability (no paid call run; verification-only; route confirmed ready)
**date:** 2026-06-30

## purpose

Run one controlled paid live canary for `starter_asymmetric_market_backed_depth` under explicit override and
capture proof the registry route works on a real asymmetric MLB game, without changing buyer semantics, artifact
schema, default allowlist, or production routing. This is the final pre-promotion gate (threshold step 5/6).

## start state

- `dai`: clean, synced, `d1fbb33` (0/0). `dai-vault`: clean, synced, `34fc2bd` (0/0).
- Pre-existing untracked `06 Execution/system-state-synopsis-v1.md` left excluded.
- `DEFAULT_ALLOWLIST` unchanged (4) and excludes asymmetric. Byte-alignment present
  (`home_form_block`/`away_form_block` in `_starter_slots`). devcore-sql up.

## outcome: BLOCKED -- no real asymmetric candidate exists in the window

The paid canary requires a real MLB game that classifies as `starter_asymmetric_market_backed_depth`:
**both starters announced, exactly one side carrying 2026 season-form quality, AND a multi-book depth market.**

Non-paid candidate search (free StatsAPI `probablePitcher` + per-pitcher `people/{id}/stats?season=2026`):

| date | announced games scanned | asymmetric-stats pairings found |
|---|---|---|
| 2026-06-30 | all | **0** |
| 2026-07-01 | 14 (1 TBD) | **0** |
| 2026-07-02 | all announced | **0** |
| 2026-07-03 | all announced | **0** |

**Every announced starter across four slates has 2026 season stats** -> every game classifies as symmetric
**enriched**, never asymmetric. The asymmetric trigger needs an announced starter with NO 2026 stats (a true MLB
debut / spot starter), which is rare in mid-season and absent in the window.

Structural conjunction problem (compounds the rarity): `market_backed_depth` requires odds posted (~1-day
horizon -> 06-30/07-01 only), while a no-stats debut starter is likeliest on a FAR slate where odds are not yet
posted (which would route `market_missing`, not `backed_depth`). The two conditions rarely co-occur.

**Decision (operator-approved):** HOLD -- document the blocker, run no paid call. A paid call now would route
`enriched` / `regime_not_allowlisted` (an invalid proof, not the asymmetric registry route), so it was not run.

## route readiness (verified non-paid -- ready to run the moment a candidate appears)

The asymmetric registry route is confirmed promotion-ready except for the live candidate:

- threshold (1) manifest valid -- **met** (`check_prompt_manifest.py` OK, 9 templates / 10 recipes).
- threshold (2) fixture canary -- **met**.
- threshold (3) byte-equivalence -- **met** (Asymmetric Overlay Byte-Alignment v1).
- threshold (4) override registry-authoritative -- **met** (fixture: promptSource=registry, fallbackReason=null,
  assembledHash 64-hex; verified by `test_canary_override_asymmetric_is_registry_authoritative`).
- threshold (5) >= 1 paid live canary -- **BLOCKED** (no real candidate).
- threshold (6) metrics route proof -- **pending** the paid run.
- threshold (7) DEFAULT_ALLOWLIST unchanged -- **held** (4 regimes, asymmetric excluded).

## exact paid command (identified, NOT executed)

When a candidate appears, the controlled paid run is:
1. Bring up FastAPI with `DAI_MLB_REGISTRY_PROMPT_CANARY=1` and
   `DAI_MLB_REGISTRY_PROMPT_CANARY_REGIMES=starter_asymmetric_market_backed_depth` (override allowlisting ONLY
   the asymmetric regime), plus `DAI_MLB_ROUTE_PROVENANCE_PATH=<scratch>` and the .NET API.
2. `POST http://localhost:5215/api/agent-runs` with
   `{runType:"sports.matchup.analysis", input:{competition:"mlb", homeTeam, awayTeam, gameDate}}` for the one
   asymmetric game -- exactly one paid gpt-4o-mini call (~$0.003), no retries.
3. Verify provenance promptSource=registry / regime=starter_asymmetric_market_backed_depth /
   regimeAllowlisted=true / legacyFallbackUsed=false / fallbackReason=null / recipe+version set / assembledHash
   64-hex; `/artifact` schema unchanged + no internal provenance leak in the buyer body; then confirm default
   (no override) still routes asymmetric to live and DEFAULT_ALLOWLIST is unchanged.

## candidate signal to watch for

Both probable pitchers announced for a NEAR (odds-posted) slate, but exactly one has zero 2026 season pitching
stats (a debut/call-up/bullpen spot start). Detect via `schedule?hydrate=probablePitcher` (both present) +
`people/{id}/stats?stats=season&group=pitching&season=2026` (splits present for one side only). Re-run this slice
when one appears; the market will be backed_depth as long as odds are posted for that game's date.

## tests / checks run (non-paid)

- `check_prompt_manifest.py` -> OK (9 templates, 10 recipes).
- `pytest tests/test_asymmetric_regime_split.py test_registry_prompt_canary.py test_prompt_route_decision.py
  test_prompt_registry_contract.py test_prompt_provenance.py` -> **99 passed** (default fails-closed, override
  registry-authoritative, byte-equivalence, symmetric unchanged, allowlist unchanged, shadow_only).

## paid-call status

**None.** Free StatsAPI candidate probes + non-paid tests only. No model call. No paid call was approved or run.

## buyer-facing impact

**None.** No code, no template/recipe/classifier/allowlist/route-key change; no .NET / DB / billing / auth /
tenant change. Verification + one new vault doc only.

## default allowlist status

**Unchanged -- exactly four regimes; `starter_asymmetric_*` excluded.** No promotion.

## risks / deferred items

- The asymmetric_market_backed_depth proof is gated on a real-world event (a debut/no-stats starter on an
  odds-posted slate) that the current schedule does not provide; timeline is event-driven, not within our control.
- Alternative if a natural candidate stays absent: accept the fixture proof (already complete, 402 tests) for
  gate 5 and promote on fixture+shadow evidence -- a policy decision deferred to the operator; not taken here.
- The route remains byte-equal + override-verified, so no code work is pending; this is purely a candidate wait.

## next recommended slice

**Outcome Reconciliation Follow-up v1** -- the non-blocked, highest-value standing work: once the 20-run live
backlog (3 soak + 8 v2 + 9 targeted) reaches Final, reconcile it to convert route coverage into performance
evidence. Re-attempt **Paid Asymmetric Canary v1** when a real asymmetric candidate appears (monitor via the
candidate signal above). If the wait is unacceptable, an operator decision on fixture-based promotion is the
alternative.
