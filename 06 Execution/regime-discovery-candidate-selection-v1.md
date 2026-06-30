# Regime Discovery + Candidate Selection v1 -- Evidence Report

**status:** complete (discovery + dry-run; no code, no paid calls, no runs)
**date:** 2026-06-30

## purpose

Design and dry-run a candidate-discovery process that deliberately finds games likely to exercise the thin /
unexercised allowlisted regimes (starter_missing_*, starter_enriched_market_missing) instead of repeatedly
capturing starter_enriched_market_backed_depth. Non-paid: free StatsAPI + odds-events probes only, read-only,
no model calls, no POST /api/agent-runs.

## start state

- `dai`: clean, synced, `0f563d6` (0/0). `dai-vault`: clean, synced, `2c133e8` (0/0).
- Pre-existing untracked `06 Execution/system-state-synopsis-v1.md` left excluded.
- `DEFAULT_ALLOWLIST` unchanged (4 regimes). Coverage matrix v1 present (9 regimes; evidence concentrated in
  enriched_market_backed_depth; starter_missing thin/unreconciled; enriched_market_missing unexercised).

## target regimes

Primary (allowlisted, under-evidenced): starter_missing_market_missing (2 live, 0 reconciled),
starter_missing_market_backed_depth (1 live, 0 reconciled), starter_enriched_market_missing (0 live).
Secondary: enriched_market_backed_depth as control/filler; the assembly_error fallback only if asymmetric
quality is detectable pre-run.

## pre-run signals (verified against source)

Regime = (starter_state x market_state). Classifier: `migration_readiness._starter_state` / `_market_state`.

- **starter_state = missing** when the .NET starter client returns a null context. `MlbStarterClient.cs:216`:
  *"both starters must be announced -- partial data is not useful"* -> if home OR away `probablePitcher` is null
  it returns `(null context, identity)`. So **>=1 TBD probable pitcher -> starter_missing** (identity/gamePk
  still preserved for reconciliation). Observable from
  `statsapi /api/v1/schedule?date=D&hydrate=probablePitcher`.
- **starter_state = enriched** when both announced AND at least one side has season-form quality (ERA etc.);
  **named** when both announced but neither has season stats (debut/rookie). The *asymmetric*-quality case
  (one side has stats, other does not) classifies as enriched but breaks the symmetric enriched recipe ->
  this is the `assembly_error` predictor (run 260018), distinct from TBD.
- **market_state = missing** when the odds provider has no event for the game (context null).
- **market_state = backed_depth** when `bookCount >= 2 AND consensusSide` set (multi-book consensus).
- **market_state = backed** when a market exists but without multi-book depth (run line only). NOTE:
  starter_missing_market_backed and the named-tier are NOT allowlisted -- avoid targeting them.

## free schedule / source probe findings

**Starter horizon (StatsAPI probablePitcher), TBD density by day-ahead lead:**

| date | lead | games | games with >=1 TBD starter |
|---|---|---|---|
| 2026-06-30 | L+0 (today) | 15 | 1 (825066, already captured) |
| 2026-07-01 | L+1 (tomorrow) | 14 | 1 (824818, already captured = soak 260015) |
| 2026-07-02 | L+2 | 9 | **6** |

**Odds horizon (the-odds-api `baseball_mlb/events`, free endpoint):** only **15 MLB events listed, ALL on the
06-30 slate** (commence 06-30 22:36 -> 07-01 01:41 UTC). **No 07-02 events have odds posted.** Odds horizon
is ~1 day.

**The key insight -- regime reachability is a function of lead time, because the two horizons differ:**

| lead | starters | odds | natural regime |
|---|---|---|---|
| L+0 / L+1 | announced | posted | enriched_market_backed_depth (the over-captured one) |
| L+2 announced game | announced (quality) | NOT posted | **starter_enriched_market_missing** |
| L+2 TBD game | >=1 TBD | NOT posted | **starter_missing_market_missing** |
| L+1 TBD game (rare) | >=1 TBD | posted | **starter_missing_market_backed_depth** |

So the thin regimes are NOT unreachable -- they are reachable by probing the **future slate (L+2) before odds
post**, which same-day capture never sees. enriched_market_missing is reachable after all (3 announced 07-02
games exist now with no odds), just not from a same-day slate.

## duplicate findings

Dedupe by (SourceProvider=mlb_statsapi, ExternalGameId/gamePk) against AgentRuns: **all 9 of the 07-02 candidate
gamePks are new** (0 existing runs). Today's/tomorrow's only TBD games (825066, 824818) are already captured and
excluded. gamePk is the stable dedupe key; the matcher already surfaces MultipleMatches if a game is captured
twice, so capturing a future game now and again near game day is allowed but must be flagged, not silent.

## candidate ranking rules

- **Priority 1** -- allowlisted regime with 0 reconciled rows and low live count
  (enriched_market_missing: 0; starter_missing_market_missing: 2; starter_missing_market_backed_depth: 1).
- **Priority 2** -- allowlisted regime with live rows but still thin.
- **Priority 3** -- known fallback-risk shape (asymmetric quality) to verify fallback attribution.
- **Priority 4** -- control rows from enriched_market_backed_depth.

## candidate table (for the NEXT paid batch -- NOT run this slice)

Run timing: capture these **while 07-02 is outside the odds horizon (i.e., on 06-30)** so the market reads
missing. All gamePks new (deduped).

| gamePk | date | away @ home | starter status | market (now) | expected regime | priority | action |
|---|---|---|---|---|---|---|---|
| 824335 | 07-02 | Marlins @ Rockies | away TBD | unposted | starter_missing_market_missing | P1 | include now |
| 824416 | 07-02 | White Sox @ Guardians | away TBD | unposted | starter_missing_market_missing | P1 | include now |
| 824906 | 07-02 | Cardinals @ Braves | home TBD | unposted | starter_missing_market_missing | P1 | include now |
| 824093 | 07-02 | Rays @ Royals | both TBD | unposted | starter_missing_market_missing | P1 | include now |
| 822884 | 07-02 | Tigers @ Rangers | home TBD | unposted | starter_missing_market_missing | P1 | include now |
| 823935 | 07-02 | Padres @ Dodgers | both TBD | unposted | starter_missing_market_missing | P1 | include now |
| 823442 | 07-02 | Pirates @ Phillies | both announced | unposted | starter_enriched_market_missing* | P1 | include now |
| 823765 | 07-02 | Reds @ Brewers | both announced | unposted | starter_enriched_market_missing* | P1 | include now |
| 823119 | 07-02 | Angels @ Mariners | both announced | unposted | starter_enriched_market_missing* | P1 | include now |

*enriched requires season-form quality on >=1 starter; if a named pitcher has no prior-season stats the game
routes `named_market_missing` (NOT allowlisted) -> confirm quality via `people/{id}/stats?season=2026` before
relying on these for the enriched_market_missing target, or treat them as best-effort.

starter_missing_market_backed_depth (allowlisted, 1 row): **watch later** -- needs a TBD game in the narrow
L+1 window where odds posted but starter still TBD; today/tomorrow offer no fresh one (824818 captured). Control
enriched_market_backed_depth filler: any same-day 07-01 announced game with posted odds.

## discovery questions answered

1. starter_missing signal: >=1 TBD `probablePitcher` (client requires both announced). 2. market_missing:
no odds event for the game. 3. market_backed_depth: bookCount>=2 AND consensusSide. 4. enriched_market_missing
unexercised because same-day enriched games always have odds. 5. Reachable? YES, via L+2 announced games before
odds post (not same-day). 6. Too late in the day? It's too *same-day* -- the lever is DAY, not hour. 7. One or
two days ahead? **Two+** (L+2 gives both TBD starters and unposted odds). 8. Dedupe sufficient? Yes by gamePk;
flag intentional re-capture. 9. Reserve slots for thin regimes + fill with control? YES -- bias the batch to
thin-regime targets, add a few enriched_backed_depth controls. 10. Approval-gate format: same as v2 (candidate
table + size + calls + cost + env + POST flow + verification plan, then WAIT).

## whether a script was added

**No script added this slice.** The manual method is now fully documented (3 steps: schedule probe with
probablePitcher over L+2..L+3 -> classify starter_state by TBD; odds-events probe -> classify market_state by
coverage; dedupe by gamePk). A tiny read-only helper `discover_regime_candidates.py` (free StatsAPI + odds
events, deterministic, prints a predicted-regime + dedupe table) is **justified** for repeated use but
**deferred** to the capture slice so it is built where it will actually run and can be validated against one
real targeted batch -- avoiding overbuild before the method is proven.

## recommended next paid batch strategy

Compose a ~6-8 run batch biased to thin regimes: 4-5 starter_missing_market_missing (from the 6 TBD 07-02
games), 2-3 starter_enriched_market_missing (from the 3 announced 07-02 games, quality-confirmed), and 1-2
enriched_market_backed_depth controls (same-day). Run on 06-30 while 07-02 odds are unposted. Gate with the v2
approval format and WAIT for explicit approval. Expected: totalRows +N, registryRows +N (or live/fallback if an
asymmetric-quality game falls back), all pending reconciliation until the games settle 07-02/07-03.

## risks / deferred items

- enriched vs named for the 3 announced games is unconfirmed without a season-stats probe; a stats-less starter
  flips them to non-allowlisted named_market_missing.
- Market state is timing-sensitive: if the batch runs after 07-02 odds post, the targets shift to
  backed_depth/enriched_backed_depth. Run while odds are unposted.
- Future-dated capture means a ~2-day-out artifact with no market -- valid but unusual; reconciliation still
  works via gamePk once the game is Final.
- starter_missing_market_backed_depth remains hard to target (narrow L+1 window); no clean candidate now.
- Point-in-time snapshot (2026-06-30); schedules/odds shift hourly.

## next recommended slice

**Targeted Live Batch Capture v1** (paid, gated) -- run the candidate table above to deliberately fill
starter_missing_market_missing and starter_enriched_market_missing, with the standard approval gate. Companion
(non-paid, time-gated): **Outcome Reconciliation Follow-up v1** once the prior 11 pending runs settle.
