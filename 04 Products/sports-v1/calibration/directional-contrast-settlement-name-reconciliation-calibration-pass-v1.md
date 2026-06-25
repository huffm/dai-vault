# Directional-Contrast Settlement + Name-Reconciliation + Calibration Pass v1

**date:** 2026-06-25
**status:** SETTLED -- 14 of the 15 frozen-manifest runs reconciled live (per-run, idempotent); 1 run (077A)
BLOCKED on a lean-encoding integrity defect. Read-only calibration pass produced. No tuning, no code change, no
prompt/model/confidence/lean/depth/advertised-strength/buyer change. Corpus 47 -> 61.
**type:** settlement + reconciliation-integrity + calibration-read slice. dai-slice-runner + dai-skill-router gate +
dai-grill-with-vault + systematic-debugging + verification-before-completion + dai-agent-handoff. dai-test-discipline
held in reserve (no code change -> no test run required).
**inputs:** frozen 15-run manifest (`pre-settlement-idempotency-duplicate-risk-manifest-v1.md`), cohort identity +
market baseline (`directional-contrast-cohort-capture-v2.md`), matcher/evaluator doctrine
(`outcome-reconciliation-matcher-v1.md`), prior read (`reconciled-outcome-calibration-read-v2.md`).

**Anchor:** Settle only what is authoritatively Final, per agentRunId, once; verify every identity against MLB
StatsAPI before writing; block anything ambiguous; then read the newly-settled cohort without tuning. The crux of
the slice turned out to be integrity, not arithmetic: the authoritative DB lean for one run contradicts the doc and
its own artifact prose.

## 1. Services and state (verified)

- repo state before == after: `dai` clean (HEAD `166bc19`, idempotency-hardening commit present locally),
  `dai-vault` clean before this doc. No code change this slice.
- Started Docker Desktop -> engine up; started container `devcore-sql` (was Exited 255) -> Up, 1433 mapped. SQL
  connectivity confirmed via `sqlcmd` (db `devcore`). Started `DevCore.Api` (Development, dev-bypass auth, tenant 1)
  on `http://localhost:5007`. FastAPI analyzer not required (reconciliation touches SQL + controller only).
- Pre-pass corpus (read-only): AgentRunOutcomes 47 / AgentRunEvaluations 47. All 15 manifest runs `completed`,
  `mlb_statsapi`, ExclusionReason null, outcome=0/eval=0 -- clean idempotency state, exactly as the manifest froze.

## 2. Authoritative outcomes (MLB StatsAPI, verified per gamePk -- never guessed)

All 14 distinct cohort gamePks are Final. Identity proof = persisted `(mlb_statsapi, gamePk)` matched to StatsAPI
away/home/officialDate + final score; no display-name matching was used for identity.

| gamePk | matchup (away @ home) | StatsAPI status | away | home | winner |
|---|---|---|---|---|---|
| 823850 | Rangers @ Marlins | Final | 2 | 4 | home |
| 823613 | Cubs @ Mets (makeup) | Final | 10 | 3 | away |
| 824583 | Guardians @ White Sox | Final | 4 | 3 | away |
| 824341 | Red Sox @ Rockies | Final | 6 | 8 | home |
| 824018 | Orioles @ Angels | Final | 6 | 7 | home |
| 824259 | Yankees @ Tigers | Final | 4 | 2 | away |
| 822965 | Royals @ Rays | Final | 3 | 5 | home |
| 823367 | Mariners @ Pirates | Final | 1 | 11 | home |
| 822719 | Phillies @ Nationals | Final | 5 | 4 | away |
| 822798 | Astros @ Blue Jays | Final | 3 | 1 | away |
| 824500 | Brewers @ Reds | Final | 6 | 5 | away |
| 823691 | Dodgers @ Twins | Final | 4 | 3 | away |
| 823041 | Diamondbacks @ Cardinals | Final | 9 | 4 | away |
| 823284 | Braves @ Padres | Final | 2 | 5 | home |

**gamePk 823613 dual-status (resolved, not blocked):** the `schedule` endpoint returns TWO entries under one
gamePk -- the original 06-22 slot (`detailedState=Postponed`, reason Rain, rescheduleDate 06-24T17:10Z, null score)
and the 06-24 makeup (`detailedState=Final`, statusCode F, away 10 / home 3, away `isWinner=true`). The makeup is
unambiguously Final; `feed/live` agreed (10-3). Settled as `away_win`. A first read that saw only the postponed
entry mislabeled it "Postponed" -- the raw JSON disambiguates it. This is the dual-entry representation of a
postponed-then-made-up game on the same gamePk, not a genuine ambiguity.

## 3. Integrity defect found -> 077A BLOCKED (the real finding)

The capture-v2 doc and the frozen manifest both record **077A (Braves @ Padres, 823284) as `null` (null
abstention)**. The authoritative denormalized `AgentRun.LeanSide` is **`home`**, and the persisted artifact's own
`LeanSide` field is also `home`. But every analytical element of the same artifact leans the **away** team:

- `Lean`: "Slight lean toward **Atlanta Braves** based on starting pitching advantage and market consensus."
- `Factors`: cite Martin Perez (the Braves' starter) and "market consensus favors the Braves."
- `CounterCase`: "The Padres could outperform..." (the counter-case is the home side -> the lean is away).

So three sources disagree on this one run's direction: **manifest = null, artifact prose/factors/market/counter-case
= away (Braves), structured LeanSide (denormalized + artifact) = home (Padres).** Reconciliation grades on the
denormalized `LeanSide`; grading 077A would record `home` vs the Padres' home win (5-2) as **`correct`**, when the
artifact's actual read was the Braves (away), who **lost**. That verdict would be a false positive injected into the
calibration corpus.

Per the hard boundaries (do not reconcile ambiguous decisions; do not guess; do not change LeanSide), **077A was
left unwritten** (outcome/eval null in DB, confirmed post-pass). A cohort-wide scan confirmed the defect is
**isolated to 077A**: the other 14 runs' artifact lean prose matches their `LeanSide` (home/away) exactly.

This is **not** a display-name reconciliation defect (identity matching used persisted gamePk only and was clean for
all 15). It is a **lean-direction encoding / denormalization integrity** defect in a single artifact. It is the
strongest reason this slice exists, and it is the recommended next-slice target.

## 4. Reconciliation executed (14 runs; per-run; idempotent)

Path used: per-run `POST /api/agent-runs/{id}/outcome` (the manifest-mandated path). The identity-keyed
`POST /reconcile` was deliberately NOT used for settlement because it keys on `(provider, gamePk)` and would collapse
the duplicate-risk pair -- proven below.

- **Pass 1 (create):** 14/14 returned `201 Created`.
- **Pass 2 (rerun same agentRunId):** 14/14 returned `409 Conflict` -- idempotency by agentRunId confirmed live, no
  duplicate or partial write.
- **`POST /reconcile` on `(mlb_statsapi, 823613)`** returned `MultipleMatches`, `evaluatedRunId=null`,
  `matchedRunIds=[beb5433e..., d879433e...]` -- it surfaced both runs and **wrote nothing**, never collapsing by
  gamePk. This is the guardrail that keeps the duplicate-risk pair safe.
- **Post-pass corpus:** AgentRunOutcomes 61 / AgentRunEvaluations 61 (was 47/47; +14). 077A outcome/eval = null.

### Duplicate-risk group G1 (gamePk 823613) -- handled per agentRunId

| agentRunId | AgentRunKey | lean | outcome written | winningSide | evalStatus |
|---|---|---|---|---|---|
| d879433e... | 220014 | home (Mets) | away_win 3-10 | away | **incorrect** |
| beb5433e... | 200018 | away (Cubs) | away_win 3-10 | away | **correct** |

Same final score, two distinct AgentRunKeys, opposite leans -> exactly one correct, one incorrect, as predicted.
The two decisions are separate evaluables on one shared game event; they are NOT two independent observations of
game direction and are kept that way in the metrics. Settling 823613 also closes the original directional-contrast
200x cohort's last open game (beb5433e / run 200018). **beb5433e remains market-baseline-missing**: outcome-only
reconciliation is SAFE (the AgentRunOutcome/Evaluation write needs no market baseline), so it was reconciled for the
DAI-vs-actual / home / chance axes and kept **out of the DAI-vs-market denominator**.

## 5. Per-run settlement verdicts (14 reconciled; from DB post-pass)

| run | gamePk | lean | outcome | verdict | regime |
|---|---|---|---|---|---|
| d379433e | 823850 | away | home_win | incorrect | cohort-v2 |
| d879433e | 823613 | home | away_win | incorrect | cohort-v2 (G1) |
| da79433e | 824583 | away | away_win | correct | cohort-v2 |
| db79433e | 824341 | away | home_win | incorrect | cohort-v2 |
| de79433e | 824018 | home | home_win | correct | cohort-v2 |
| e379433e | 824259 | home | away_win | incorrect | cohort-v2 |
| e979433e | 822965 | home | home_win | correct | cohort-v2 |
| ee79433e | 823367 | home | home_win | correct | cohort-v2 |
| f579433e | 822719 | away | away_win | correct | cohort-v2 |
| f779433e | 822798 | home | away_win | incorrect | cohort-v2 |
| fe79433e | 824500 | home | away_win | incorrect | cohort-v2 (mkt disagreement) |
| 037a433e | 823691 | away | away_win | correct | cohort-v2 |
| 047a433e | 823041 | home | away_win | incorrect | cohort-v2 |
| beb5433e | 823613 | away | away_win | correct | 200x (market-missing) |
| 077a433e | 823284 | (home, contested) | -- BLOCKED -- | -- | cohort-v2 (deferred) |

## 6. Calibration pass (read-only; do not over-interpret; n is tiny)

**Counts.** Runs considered: 15. Reconciled today: 14. Excluded: 1 (077A, lean-encoding contradiction). With market
baseline: 13. Without market baseline: 1 (beb5433e). Null/abstain reconciled: 0 (077A was contested, not a clean
null, and is blocked). Unresolved: 0.

**DAI lean result (14 reconciled):** hit 7, miss 7, null/abstain 0, unresolved 0.
**Cohort-v2 slate only (13 reconciled, the comparable directional set):** hit 6, miss 7 -> **46.2%** on decided
leans. (beb5433e, the 200x run, is kept separate: 1/1 correct.)

**DAI vs market favorite (13 cohort-v2 reconciled; market present for all 13):**
- DAI matched the market favorite: **12**. DAI disagreed: **1** (fe79, Brewers @ Reds: DAI home/Reds, market
  away/Brewers). DAI null/abstain: 0. Market baseline missing: 0 (within cohort-v2; beb5433e is the only missing one
  and sits in the 200x regime).
- On the 12 agreements DAI == market by construction: **6/12 = 50%**. On the 1 disagreement, **market was right and
  DAI was wrong** (Brewers won). Net: DAI **6/13 (46.2%)** vs market **7/13 (53.8%)** on the same decided set -- the
  whole gap is the single disagreement.

**Home/away lean distribution (cohort-v2 reconciled, 13):** home 8 / away 5. Home-lean accuracy **3/8 = 37.5%**;
away-lean accuracy **3/5 = 60%**.
**Outcome direction this slate (13 distinct reconciled games):** home wins 5 / away wins 8 -> **road-heavy** (62%
away). A naive always-lean-home baseline scores **5/13 = 38.5%**; DAI's 46.2% is marginally above it (DAI took 5
away leans, 3 hit).

**Confidence distribution (cohort-v2 reconciled):** 0.80 x4, 0.75 x9. The four 0.80 runs went **0/4 (0%)**; the
0.75 runs went **6/9 (66.7%)**. n=4 at 0.80 is noise, but it reinforces -- it does not contradict -- read v2:
confidence is near-constant and does NOT positively rank outcomes here.
**Advertised strength:** High x14 (uniform; no discrimination). **Evidence richness:** 2 x14. **Source
sufficiency:** moderate x14 (uniform regime; no thin/enriched contrast).

**Duplicate-risk groups:** one (G1), handled per agentRunId (sec 4). **Display-name normalization issues:** none at
the identity layer (persisted gamePk only). One lean-encoding integrity defect (077A, sec 3).

### Is this cohort calibration-useful yet? No (for tuning).

- n=13 cohort-v2 decided, ONE slate, ONE market disagreement. Below the read-v2 target (~20-30 decided away leans,
  multiple slates). This is slate 2 of the planned 2-3.
- DAI mostly tracks the market favorite (12/13 agree); on the one game where it diverged, the market won. The
  independent-signal sample is n=1 and negative so far -- **no independent directional edge is demonstrated.**
- On a road-heavy slate DAI's home skew (8 home leans) underperformed (37.5%), consistent with read v2's "leans
  home, scores about the base rate." Outcome accuracy (46.2%) is barely above the always-home baseline (38.5%) and
  well inside binomial noise at n=13.
- Confidence remains non-discriminating. Advertised strength / richness / sufficiency are uniform -> no
  within-cohort contrast to read.

**Prior conclusion preserved (evidence did not change it):** calibration is NOT proven; there is still no validated
directional-discrimination target; **tuning is premature.** This slate adds mild reinforcement to "no edge
demonstrated," not a conclusion in either direction.

## 7. What did not change

No prompt, model, confidence, lean, posture, source-depth, advertised-strength, buyer-copy, market-baseline
derivation, Tool Gateway, reconciliation logic, or frontend change. No LeanSide was altered (077A left as-is and
unsettled). No outcomes guessed. No code change (existing per-run idempotent path was sufficient and was proven, not
modified). Only operational data was written: 14 AgentRunOutcomes + 14 AgentRunEvaluations.

## 8. Verification

- StatsAPI: every gamePk fetched live (`/schedule` + `/linescore` + `/feed/live` cross-checks); 823613 disambiguated
  from raw JSON. No score inferred.
- Reconciliation: Pass 1 14x201, Pass 2 14x409, `/reconcile` 823613 MultipleMatches/no-write -- all observed live.
- DB post-pass: corpus 61/61; 14 verdicts read back and matched predictions; 077A outcome/eval null (blocked).
- Repo: `dai` and `dai-vault` clean (no code change); this doc is the only file added.

## 9. Recommended next slice (no tuning)

1. **Reconciliation Lean-Encoding Integrity v1** (highest value): investigate why 077A's denormalized + artifact
   `LeanSide` (`home`) contradicts its own prose/factors/market/counter-case (away/Braves) and why the manifest read
   it as `null`. Determine whether the analyzer mis-serialized LeanSide or the denormalization inverted it; add a
   targeted test (artifact prose/lean direction vs persisted LeanSide consistency) and a settlement-time guard that
   refuses to grade a run whose artifact lean direction disagrees with its denormalized LeanSide. Then decide 077A's
   true verdict and settle or exclude it explicitly.
2. **Directional-Contrast Cohort Capture v3 + Market Baseline Capture v2** (2026-06-25/-26 slate): still need 1-2
   more slates and more DAI-vs-market disagreements before any directional conclusion.
3. **Reconciled-Outcome Calibration Read v3**: defer until >=3 settled market-baseline-backed slates are pooled;
   keep regimes separate (never pool 180x/190x/200x/cohort-v2).
