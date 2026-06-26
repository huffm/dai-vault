# Directional-Contrast Cohort Capture v3 + Market Baseline v2

**date:** 2026-06-26
**status:** CAPTURED -- 15 identity-bearing, market-backed MLB runs generated pregame. UNSETTLED (games settle
~2026-06-27T04:00Z+). No tuning, no code/prompt/confidence/depth/advertised-strength/buyer/settlement/reconciliation/
calibration change. No Cognitive Factory activation.
**type:** Lane A evidence-acquisition slice. Generation + market-baseline collection + artifact trace only.
dai-slice-runner + dai-skill-router gate + verification-before-completion. Read-only against settlement/calibration.
**anchor:** Capture a fresh, identity-bearing, market-backed cohort so a future settlement agent can compare DAI vs
market vs outcome. Strengthens Evidence Readiness Gate 3/4 evidence base. No production decision logic modified.

See also: [[calibration-assessment-v3]] (59-run integrity-qualified baseline), [[directional-contrast-cohort-capture-v2]]
(predecessor slate, 2026-06-24), Evidence Readiness Gates, [[prompt-ledger-hook-v1]].

## 1. Repo / service preflight (verified)

| item | state |
|---|---|
| dai HEAD | `e629bac` (main, clean, == origin/main; ff-only pull: already up to date) |
| dai-vault HEAD (pre-doc) | `ed3ec7d` (main, clean, == origin/main; ff-only pull: already up to date) |
| pipeline freshness | DevCore.Api built + run from HEAD `e629bac` (`dotnet run`, cold build, not a stale binary) |
| Docker / devcore-sql | docker 29.1.3; container `devcore-sql` up, port 1433; db `devcore` present |
| FastAPI analyzer :8000 | healthy (`/api/ping` -> `{"status":"ok"}`); route `/api/sports/analyze` registered |
| DevCore.Api :5007 | healthy (`/health` -> `{"status":"ok"}`); listening from current HEAD |
| MLB StatsAPI | reachable; 15 games on 2026-06-26 |
| Odds API (the-odds-api v4) | reachable; 15 MLB events; 377 requests remaining after capture |
| Cognitive Factory isolation | INERT. No `CognitiveFactory` config section in either appsettings -> code defaults are off
(`CognitiveFactoryActivationOptions`: master/execution/gateway/mutation all default off, "NO runtime path consumes
these values yet"). No diagnostics or execution endpoint called. Stage 2 audit-only execution NOT invoked. |

The normal sports pipeline (`POST /api/agent-runs` -> AgentRunService -> retrieve/analyze/evaluate/compose) is wholly
separate from the Cognitive Factory; no activation path was touched.

## 2. Slate selection (pregame-only)

Capture time 2026-06-26T12:57Z (08:57 ET). All 15 StatsAPI games were in `Preview` (not started), earliest first
pitch 2026-06-26T22:40Z (~582 min out). **15 included, 0 excluded.** Every included game carries StatsAPI gamePk +
status + scheduled start and a matched odds event id + commence time. No game was inferred; identities traced to
StatsAPI and the-odds-api independently.

- included games: 15 (all pregame, all market-matched)
- excluded games: 0
- no game started before its run was captured

## 3. Cohort run manifest (15 runs, all 2026-06-26, RunType sports.matchup.analysis, artifact v3)

Lean column shows persisted `AgentRun.LeanSide` vs artifact `leanSide`; `OK` = they match (verified per run via DB
read + artifact read-back). `envs` = source-envelope count; `depth` = per-signal depth/status from the artifact.

| run | matchup (away @ home) | gamePk | lean (persisted=artifact) | conf | posture | advStr | rich | srcSuff | envs | consistency |
|---|---|---|---|---|---|---|---|---|---|---|
| 4d04433e | Houston Astros @ Detroit Tigers | 824255 | home (OK) | 0.75 | monitor | High | 2 | moderate | 4 | Consistent |
| 5104433e | Cincinnati Reds @ Pittsburgh Pirates | 823363 | home (OK) | 0.80 | monitor | High | 2 | moderate | 4 | Consistent |
| 5704433e | Washington Nationals @ Baltimore Orioles | 824824 | home (OK) | 0.75 | monitor | High | 2 | moderate | 4 | Consistent |
| 5c04433e | Texas Rangers @ Toronto Blue Jays | 822796 | away (OK) | 0.75 | monitor | High | 2 | moderate | 4 | Consistent |
| 6304433e | Seattle Mariners @ Cleveland Guardians | 824423 | home (OK) | 0.75 | monitor | High | 2 | moderate | 4 | Consistent |
| 6a04433e | Arizona Diamondbacks @ Tampa Bay Rays | 822962 | home (OK) | 0.80 | monitor | High | 2 | moderate | 4 | Consistent |
| 7104433e | Philadelphia Phillies @ New York Mets | 823610 | null (OK) | 0.54 | monitor | Medium | 1 | thin | 4 | NotEvaluable |
| 7704433e | New York Yankees @ Boston Red Sox | 824743 | home (OK) | 0.75 | monitor | High | 2 | moderate | 4 | Consistent |
| 7c04433e | Kansas City Royals @ Chicago White Sox | 824582 | home (OK) | 0.675 | monitor | Medium | 1 | thin | 4 | Consistent |
| 8304433e | Chicago Cubs @ Milwaukee Brewers | 823773 | home (OK) | 0.80 | monitor | High | 2 | moderate | 4 | Consistent |
| 8704433e | Colorado Rockies @ Minnesota Twins | 823688 | home (OK) | 0.75 | monitor | High | 2 | moderate | 4 | Consistent |
| 8c04433e | Miami Marlins @ St. Louis Cardinals | 823039 | home (OK) | 0.75 | monitor | High | 2 | moderate | 4 | Consistent |
| 8d04433e | Athletics @ Los Angeles Angels | 824014 | away (OK) | 0.75 | monitor | High | 2 | moderate | 4 | Consistent |
| 9404433e | Los Angeles Dodgers @ San Diego Padres | 823285 | away (OK) | 0.75 | monitor | High | 2 | moderate | 4 | Consistent |
| 9704433e | Atlanta Braves @ San Francisco Giants | 823206 | away (OK) | 0.63 | monitor | Medium | 1 | thin | 4 | Consistent |

Run id shown is the first 8 chars of the AgentRunId; full ids + AgentRunKey + CorrelationId are in the machine
manifest (sec 7). All 15: `completed`, `mlb_statsapi` provider, `ExclusionReason IS NULL`, tenantKey 1, posture
`monitor`, artifact `sports_decision_artifact_v3`, 4 source envelopes each.

## 4. Cohort summary

- **Lean distribution: home 10, away 4, null 1.** (v2 was 8 home / 5 away / 1 null on a 7/7 market-balanced slate;
  this slate's market is home-favorite-heavy -- see sec 5 -- and DAI tracks it.)
- **Confidence:** decided runs cluster tightly (10 of 14 at 0.75-0.80); the three thin runs sit lower (0.54, 0.63,
  0.675). Still low-variance among the enriched majority; unlikely to discriminate, consistent with [[calibration-assessment-v3]]
  ("confidence is not predictive").
- **Directional contrast is present this cohort** (the v3 design goal v2 lacked): **12 enriched runs** (richness 2,
  advertised High, `moderate` sufficiency, starter + market both `enriched/observed`) vs **3 thin runs** (richness 1,
  advertised **Medium**, `thin` sufficiency, `starting_pitching: none/missing`). The thin runs are Phillies@Mets
  (null), Royals@White Sox (home), Braves@Giants (away) -- games with no probable starter published at capture time.
  This gives a real enriched-vs-thin depth contrast within one slate.
- **Source envelopes (new in v3 vs v2):** every run carries 4 envelopes -- `starting_pitching`, `market`,
  `availability` (none/missing this slate), `bullpen_fatigue` (shallow/proxy). v2 cohorts exposed only starter +
  market; v3 surfaces availability and bullpen_fatigue envelopes explicitly (missing/proxy stay honestly visible).
- **Advertised strength:** High x12, Medium x3 (the thin-cap fired exactly on the three richness-1 runs).

## 5. Market baseline v2 (de-vigged consensus moneyline)

Source: the-odds-api.com v4 `baseball_mlb` h2h, us region, american odds, captured **2026-06-26T12:57:07Z** pregame.
Per-book two-way de-vig (normalize home+away implied to 1.0), consensus = mean across books. `home$`/`away$` =
median american price across books; `impl` = vig-included consensus; `devig` = de-vigged consensus.

| run | matchup | oddsEventId | books | home$ | away$ | impl H/A | devig H/A | favorite | favProb | DAI lean | agree? |
|---|---|---|---|---|---|---|---|---|---|---|---|
| 4d04433e | Houston Astros @ Detroit Tigers | a46f3826 | 9 | -118 | +100 | 0.537/0.501 | 0.517/0.483 | home | 0.517 | home | yes |
| 5104433e | Cincinnati Reds @ Pittsburgh Pirates | fadbcf06 | 9 | -200 | +168 | 0.667/0.372 | 0.642/0.358 | home | 0.642 | home | yes |
| 5704433e | Washington Nationals @ Baltimore Orioles | b15cd27e | 9 | -136 | +118 | 0.580/0.459 | 0.558/0.442 | home | 0.558 | home | yes |
| 5c04433e | Texas Rangers @ Toronto Blue Jays | 08d5e50d | 9 | -105 | -112 | 0.512/0.526 | 0.493/0.507 | away | 0.507 | away | yes |
| 6304433e | Seattle Mariners @ Cleveland Guardians | 9637bef3 | 9 | -112 | -105 | 0.527/0.511 | 0.508/0.492 | home | 0.508 | home | yes |
| 6a04433e | Arizona Diamondbacks @ Tampa Bay Rays | cf96e2e4 | 9 | -136 | +119 | 0.581/0.458 | 0.559/0.441 | home | 0.559 | home | yes |
| 7104433e | Philadelphia Phillies @ New York Mets | f950765d | 9 | +139 | -162 | 0.420/0.618 | 0.405/0.595 | away | 0.595 | null | - |
| 7704433e | New York Yankees @ Boston Red Sox | 6eb5de1a | 9 | -103 | -114 | 0.505/0.534 | 0.486/0.514 | away | 0.514 | home | **NO** |
| 7c04433e | Kansas City Royals @ Chicago White Sox | fbe3805e | 6 | -132 | +110 | 0.570/0.473 | 0.546/0.454 | home | 0.546 | home | yes |
| 8304433e | Chicago Cubs @ Milwaukee Brewers | f2bbab66 | 9 | -270 | +220 | 0.729/0.310 | 0.701/0.299 | home | 0.701 | home | yes |
| 8704433e | Colorado Rockies @ Minnesota Twins | dc2d29e6 | 9 | -165 | +140 | 0.625/0.413 | 0.602/0.398 | home | 0.602 | home | yes |
| 8c04433e | Miami Marlins @ St. Louis Cardinals | 10965e1f | 9 | -114 | -103 | 0.532/0.507 | 0.512/0.488 | home | 0.512 | home | yes |
| 8d04433e | Athletics @ Los Angeles Angels | c2d68e46 | 9 | +107 | -123 | 0.485/0.553 | 0.467/0.533 | away | 0.533 | away | yes |
| 9404433e | Los Angeles Dodgers @ San Diego Padres | 136805c6 | 9 | +125 | -145 | 0.447/0.591 | 0.431/0.569 | away | 0.569 | away | yes |
| 9704433e | Atlanta Braves @ San Francisco Giants | ffc9895e | 4 | +104 | -122 | 0.490/0.549 | 0.472/0.528 | away | 0.528 | away | yes |

- **Market favorite distribution: home 9 / away 6** (home-favorite-heavy slate). All low-conviction (favProb
  0.507-0.701, mostly 0.51-0.60), typical MLB; only Cubs@Brewers (0.701) and Phillies@Mets (0.595) are clear.
- **All 15 market-backed; 0 missing.** Book coverage 9/game except Royals@White Sox (6) and the late Braves@Giants (4).

## 6. Headline observation (report only -- NO outcomes, do not over-interpret)

**DAI's lean tracks the market favorite on 13 of 14 decided-lean games (93%).**

- Agreements: 13. Lone disagreement: **NY Yankees @ Boston Red Sox** -- DAI leaned home (Red Sox), market favored
  away (Yankees) at a near-coin-flip devig 0.514. DAI abstained (null) on **Phillies @ Mets**, where the market had
  its second-clearest favorite (away/Mets 0.595) -- i.e. DAI declined a read precisely where a starter was missing.
- This reproduces the v2 finding ([[directional-contrast-cohort-capture-v2]]: 12/13, 92%): DAI's directional signal
  is largely market-derived. The discriminating sample is again tiny -- 1 disagreement + 1 null abstention this slate.
- Implication unchanged: if DAI mostly echoes the market it cannot structurally beat it; any edge must come from the
  disagreements, the null abstentions, and confidence. n=14 decided, one slate -- an observation to pool, not a conclusion.

## 7. Artifact traceability (read-back verified)

Machine-readable manifest (full AgentRunIds, AgentRunKey, CorrelationId, per-run market block, per-run integrity
flags, capture metadata): `04 Products/sports-v1/calibration/artifacts/20260626-directional-contrast-cohort-v3-manifest.json`.

Per-run read-back verified for all 15: (a) DB row exists in `dbo.AgentRuns` (persisted LeanSide, ExternalGameId,
SourceProvider, ScheduledStartUtc, ExclusionReason NULL); (b) artifact reads 200 via `GET /api/agent-runs/{id}/artifact`;
(c) artifact `externalGameId` == StatsAPI gamePk; (d) persisted `LeanSide` == artifact `leanSide`; (e) market
baseline attached by gamePk + odds event id. The manifest is sufficient for a settlement agent to reconcile each run
by gamePk -> StatsAPI Final score -> `POST /api/agent-runs/{id}/outcome` (graded on persisted LeanSide) without guessing.

## 8. Integrity scan (fresh cohort)

Ran the lean/artifact integrity scan over all 15 runs. **Result: CLEAN. Zero PotentialMismatch.**

- persisted `LeanSide` vs artifact `leanSide`: **15/15 match.**
- artifact direction consistency (`artifactDirectionConsistency`, the Decision Encoding Integrity guard checking
  lean/summary/factors prose against the structured side): 14 `Consistent`, 1 `NotEvaluable` (Phillies@Mets, null
  lean -- correct: a null lean has no side to check; not a mismatch). Zero warnings on any run.
- game identity present: 15/15 (`mlb_statsapi` + gamePk, persisted + artifact agree).
- artifact version present: 15/15 (`sports_decision_artifact_v3`).
- source envelopes present: 15/15 (4 each).
- duplicate same-game runs: none (15 distinct gamePks).
- market baseline linkage: 15/15 valid.

No mismatch to flag, settle, or remediate. No tuning. (Contrast: prior remediated mismatches in [[directional-contrast-cohort-capture-v2]]
and the mismatch-remediation history are unaffected by this read-only capture.)

## 9. Prompt-ledger capture

Capture mode: pre-execution (the finalized slice prompt is capture-worthy per [[prompt-ledger-hook-v1]] sec B --
it initiates a slice and governs calibration evidence). `OBSIDIAN_PROMPT_LEDGER_ROOT` is **unset** and there is no
agreed in-vault ledger folder, so per the hook's fallback ("if the ledger root is unavailable, embed the record
rather than fabricating an external path") **no external prompt-ledger file was written.** Compact embedded record:

- date 2026-06-26; project dai; slice `directional-contrast-cohort-v3-market-baseline-v2`; lane calibration;
  activation_stage none; prompt_type calibration; agent_target claude; capture_mode pre-execution; status executed;
  source chat-prompt (`/remote-control`); repo_target dai-vault; related_docs this file + the manifest json.
- guardrails: no tuning / no settlement / no Cognitive Factory activation / no runtime behavior change / no code change.

## 10. Settlement readiness and what remains before Calibration Assessment v4

- **Cohort is settlement-ready (UNSETTLED now).** Every valid run has identity + artifact traceability; every
  market-comparable run (all 15) has Market Baseline v2 fields. Games settle ~2026-06-27T04:00Z+; reconcile 2026-06-27.
- **NOT settled this slice.** No outcomes/evaluations written; no scores guessed; corpus settled totals unchanged.
- **Cohort size adequacy: still inadequate alone.** 15 runs, 14 decided, 1 disagreement, 1 null -- one slate. Per
  [[calibration-assessment-v3]] and v2, pool >=3 settled slates before any directional/market conclusion. The
  DAI~market agreement (13/14) keeps the discriminating sample scarce.
- **Before Calibration Assessment v4:** settle this slate (2026-06-27), keep the directional-contrast cohorts in
  their own regime (do not pool with 180x/190x/200x), and only then read DAI vs market vs outcome with the
  disagreement set (Yankees@Red Sox) and null-abstention set (Phillies@Mets) called out separately. Denominator
  doctrine unchanged: settled AND `ExclusionReason IS NULL`. **Do NOT tune** confidence/depth/lean -- no validated
  discrimination target exists yet.

## 11. No tuning / no change (held)

No prompt, confidence, source-depth, advertised-strength, buyer-copy, Tool Gateway, settlement, reconciliation,
calibration-logic, parser, posture, model, or Cognitive Factory change. Only evidence was captured: 15 new agent-run
rows + an external market baseline + an artifact trace manifest. Nothing was optimized.
