# Directional-Contrast Cohort Capture v2 + Market Baseline Capture v1

**date:** 2026-06-24
**status:** CAPTURED -- 14 identity-bearing MLB runs generated + per-game market baseline recorded. UNSETTLED
(games settle ~2026-06-25T04:00Z+). No tuning, no code/prompt/confidence/depth/advertised-strength/buyer change.
**type:** experiment-capture slice. Generation + market-baseline collection only. dai-slice-runner + dai-skill-router
gate + superpowers:verification-before-completion.

**Anchor:** Capture evidence to test whether DAI provides information beyond (1) home base rates, (2) chance, and
(3) the closing market favorite. No production decision logic was modified. If results look bad, change nothing.

## 1. Design and honest constraints

- **Single slate available.** The upcoming endpoint exposes only today's slate (2026-06-24, 15 MLB games). True
  multi-slate coverage (a v2 design goal) cannot be completed in one session -- this is **slate 1 of N**. Follow-up
  captures on 2026-06-25 and 2026-06-26 are required to span home-favorable and road-favorable slates (see sec 6).
- **Balance is observed, not forced.** DAI decides the lean; the operator only selects games. Actual distribution
  is recorded (sec 3). On this slate the market was balanced (7 home / 7 away favorites), and DAI's leans were far
  more balanced than the historical 82% home (8 home / 5 away / 1 null).
- **One capture failed.** Athletics @ San Francisco Giants (a late west-coast game, ambiguous franchise naming)
  hung in `pending` with no artifact; excluded as `invalid`. Cohort finalized at 14.

## 2. Cohort manifest (14 identity-bearing runs; all 2026-06-24, RunType sports.matchup.analysis)

| run | matchup (away @ home) | lean | conf | advStr | richness | srcSuff | gamePk | provider | mkt favorite | mkt favProb |
|---|---|---|---|---|---|---|---|---|---|---|
| d379433e | Texas Rangers @ Miami Marlins | away | 0.80 | High | 2 | moderate | 823850 | mlb_statsapi | away | 0.524 |
| d879433e | Chicago Cubs @ New York Mets | home | 0.75 | High | 2 | moderate | 823613 | mlb_statsapi | home | 0.513 |
| da79433e | Cleveland Guardians @ Chicago White Sox | away | 0.75 | High | 2 | moderate | 824583 | mlb_statsapi | away | 0.512 |
| db79433e | Boston Red Sox @ Colorado Rockies | away | 0.80 | High | 2 | moderate | 824341 | mlb_statsapi | away | 0.605 |
| de79433e | Baltimore Orioles @ Los Angeles Angels | home | 0.75 | High | 2 | moderate | 824018 | mlb_statsapi | home | 0.533 |
| e379433e | New York Yankees @ Detroit Tigers | home | 0.80 | High | 2 | moderate | 824259 | mlb_statsapi | home | 0.550 |
| e979433e | Kansas City Royals @ Tampa Bay Rays | home | 0.75 | High | 2 | moderate | 822965 | mlb_statsapi | home | 0.575 |
| ee79433e | Seattle Mariners @ Pittsburgh Pirates | home | 0.75 | High | 2 | moderate | 823367 | mlb_statsapi | home | 0.516 |
| f579433e | Philadelphia Phillies @ Washington Nationals | away | 0.75 | High | 2 | moderate | 822719 | mlb_statsapi | away | 0.547 |
| f779433e | Houston Astros @ Toronto Blue Jays | home | 0.80 | High | 2 | moderate | 822798 | mlb_statsapi | home | 0.587 |
| fe79433e | Milwaukee Brewers @ Cincinnati Reds | home | 0.75 | High | 2 | moderate | 824500 | mlb_statsapi | **away** | 0.553 |
| 037a433e | Los Angeles Dodgers @ Minnesota Twins | away | 0.75 | High | 2 | moderate | 823691 | mlb_statsapi | away | 0.615 |
| 047a433e | Arizona Diamondbacks @ St. Louis Cardinals | home | 0.75 | High | 2 | moderate | 823041 | mlb_statsapi | home | 0.519 |
| 077a433e | Atlanta Braves @ San Diego Padres | **null** | 0.75 | High | 2 | moderate | 823284 | mlb_statsapi | away | 0.525 |

Market baseline = de-vigged consensus moneyline (the-odds-api.com, ~9 books/game, h2h) captured 2026-06-24 pregame.
favProb is the favorite's de-vigged implied probability; all are low-conviction (0.51-0.62), typical MLB.

## 3. Cohort summary

- **Lean distribution: home 8, away 5, null 1** (much more balanced than the historical 82% home).
- **Confidence:** avg 0.764; 10 runs at 0.75, 4 at 0.80. Still tightly clustered (no run below 0.75); confidence
  remains low-variance and unlikely to discriminate (consistent with Read v2).
- **Advertised strength:** High x14 (all; richness 2 -> no thin cap fired).
- **Evidence richness:** 2 x14. **Source sufficiency:** moderate x14. (Uniform regime -- starter identity +
  market, no thin and no enriched-depth contrast in this cohort.)
- **Market favorite distribution:** home 7 / away 7 (balanced slate -- ideal for a clean DAI-vs-market test).

## 4. Headline observation (report only, do not over-interpret)

**DAI's lean tracks the market favorite on 12 of 13 decided-lean games (92%).**

- Agreements: 12. Lone disagreement: **Milwaukee Brewers @ Cincinnati Reds** -- DAI leaned home (Reds), market
  favored away (Brewers, favProb 0.553). DAI abstained (null) on **Braves @ Padres** (market favored away).
- This reframes the Read v2 "82% home" finding: on a market-balanced slate (7/7), DAI was 8 home / 5 away -- so the
  prior home skew looks like **market-following on home-favorite-heavy historical slates, not an intrinsic home
  bias**. DAI's directional signal appears largely market-derived.
- Implication for the experiment: if DAI mostly echoes the market, it structurally cannot beat the market; any
  edge must come from the disagreements (1 here), the null abstentions (1 here), and confidence. A single slate
  has almost no disagreement sample -- this is the central reason more slates are required.
- Caveat: one slate, n=13 decided leans. This is an observation to confirm or refute across slates 2-3, not a
  conclusion.

## 5. Market comparison readiness (verified)

Every field needed for the post-settlement read is recorded per run: DAI LeanSide, Confidence, AdvertisedStrength,
gamePk (-> authoritative outcome via MLB StatsAPI), market favorite, and market implied probability. After
settlement, Calibration Read v3 can compute:

- **DAI accuracy** = DAI lean vs winner, over the 13 decided-lean runs.
- **Market accuracy** = market favorite vs winner, over all 14 (the market has a favorite for every game).
- **DAI minus market delta** = DAI hit rate - market hit rate (on the common decided set), plus the head-to-head on
  the 1 disagreement and the 1 DAI-null game.
- **Beyond-base-rate / beyond-chance**: DAI hit rate vs the home-win base rate on this slate, and vs a 50% null.

This satisfies the success criterion: the cohort can answer (1) beats home bias? (2) beats random? (3) beats the
closing market favorite? -- once settled (and pooled across slates for power).

## 6. Identity and reconciliation eligibility (verified)

- All 14 runs are `completed`, identity-bearing (`mlb_statsapi` + gamePk), active (`ExclusionReason=null`),
  `sports_decision_artifact_v3`. Reconcilable via `POST /api/agent-runs/{id}/outcome` the moment each game is Final,
  graded on the persisted denormalized `AgentRun.LeanSide`.
- **gamePk 823613 collision (noted):** today's Cubs @ Mets run `d879433e` shares gamePk 823613 with the postponed
  Directional-Contrast cohort run `200018` (`beb5433e`, away lean) -- it is the same rescheduled game, now playing
  2026-06-24. The two runs hold OPPOSITE leans (200018 away, d879 home). Reconcile both per-run (not by identity, to
  avoid MultipleMatches); settling 823613 also closes the directional cohort's last open game.

## 7. Settlement timing, adequacy, and readiness for Read v3

- **Settlement timing:** today's games settle ~2026-06-25T04:00Z+; reconcile 2026-06-25. The late games
  (Braves@Padres, and the failed Athletics@Giants) settle 2026-06-25.
- **Cohort size adequacy: NOT YET ADEQUATE.** 14 runs, 13 decided leans, 5 away leans, 1 market disagreement, ONE
  slate. Read v2's target (~20-30 decided away leans, multiple slates) is not met. This cohort alone cannot conclude
  on directional discrimination or market-beating; the DAI~market agreement means the discriminating sample (DAI
  vs market disagreements) is tiny.
- **Readiness for Calibration Read v3:** capture 2-3 more daily slates (2026-06-25, -26, optionally -27) the same
  way, settle each, then pool for Read v3. Read v3 should report DAI vs market vs outcome with the disagreement set
  and null-abstention set called out separately, kept in its own regime (never pooled with 180x/190x/200x).

## 8. No tuning (held)

No prompt, confidence, source-depth, advertised-strength, buyer-copy, Tool Gateway, reconciliation-logic, or model
change. Only evidence was captured: 14 new agent-run rows + an external market baseline. Nothing was optimized.

## 9. Recommendation (settlement timing / adequacy / Read v3 readiness only; NO tuning)

1. **Settlement timing:** reconcile this cohort 2026-06-25 after games are Final (authoritative StatsAPI scores).
2. **Cohort size adequacy:** inadequate alone; capture 2-3 more daily slates before any directional/market
   conclusion. The DAI~market agreement (12/13) makes additional disagreement samples the scarce, essential data.
3. **Readiness for Read v3:** ready to design; run it only after >=3 settled slates are pooled. Do NOT tune
   confidence, depth, or lean -- there is still no validated discrimination target.

## 10. Verification

- Generation: 15 POSTs to `/api/agent-runs` (RunType sports.matchup.analysis); 14 completed identity-bearing, 1
  (Athletics@Giants) hung and was excluded `invalid`. Each artifact read live for lean/confidence/advStr/richness/
  sufficiency/gamePk/provider.
- Market baseline: the-odds-api.com h2h consensus (us region, ~9 books/game), de-vigged to favorite implied
  probability, captured pregame 2026-06-24. Saved per matchup.
- Identity: all 14 carry mlb_statsapi + gamePk; confirmed active and completed.
- Outcomes: NOT captured this slice (games unsettled; no scores guessed). Corpus settled totals remain 47/47.
- No code/prompt/model/confidence/depth/advertised-strength/buyer/Tool-Gateway/reconciliation change.
