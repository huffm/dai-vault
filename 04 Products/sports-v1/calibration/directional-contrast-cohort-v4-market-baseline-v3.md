# Directional-Contrast Cohort Capture v4 + Market Baseline v3

**date:** 2026-06-28
**status:** CAPTURED -- 15 identity-bearing, market-backed MLB runs generated pregame. UNSETTLED (games settle
~2026-06-29T04:00Z+). No tuning, no code/prompt/confidence/depth/advertised-strength/buyer/settlement/reconciliation/
calibration change. No Cognitive Factory activation.
**type:** Lane B evidence-acquisition slice. Generation + market-baseline collection + artifact integrity trace only.
dai-slice-runner + dai-skill-router gate + verification-before-completion. Read-only against settlement/calibration.
**anchor:** Capture a fresh, identity-bearing, market-backed cohort so a future settlement agent can compare DAI vs
market vs outcome. Strengthens Evidence Readiness Gate 3/4 evidence base. No production decision logic modified.

See also: [[calibration-assessment-v3]] (59-run integrity-qualified baseline -> 74 after v3 settlement),
[[directional-contrast-cohort-v3-market-baseline-v2]] (predecessor slate, 2026-06-26, now SETTLED -- see the v3
settlement note), [[mismatch-remediation-v1]] (denominator + PotentialMismatch doctrine), Evidence Readiness Gates.

## 1. Repo / service preflight (verified)

| item | state |
|---|---|
| dai HEAD | `fee495d` (main, clean, == origin/main) |
| dai-vault HEAD (pre-doc) | `6218aab` (main) |
| pipeline freshness | DevCore.Api built + run from HEAD `fee495d` (`dotnet run`, cold build); /health ok |
| Docker / devcore-sql | container `devcore-sql` up, port 1433; db `devcore` present |
| FastAPI analyzer :8000 | healthy (`/api/ping` -> `{"status":"ok"}`); `/api/sports/analyze` registered; OpenAI key loaded (no 429) |
| DevCore.Api :5007 | healthy (`/health` -> ok); dev bypass auth -> tenantKey 1 |
| MLB StatsAPI | reachable; 15 games on 2026-06-28, all `Scheduled` (pregame) at capture |
| Odds API (the-odds-api v4) | reachable; 15 MLB h2h events, 9 books each |

The normal sports pipeline (`POST /api/agent-runs` -> retrieve(statsapi+odds)/analyze/evaluate/compose) was used;
no Cognitive Factory path touched (no `CognitiveFactory` config section -> defaults off).

## 2. Slate selection (pregame-only)

All 15 StatsAPI games for 2026-06-28 were `Scheduled` (not started) at capture; earliest first pitch 2026-06-28T17:35Z.
**15 included, 0 excluded.** Each run carries StatsAPI gamePk + a matched odds event. One run per gamePk (no
duplicate same-game runs): the Astros@Tigers run was captured first as the pipeline/OpenAI health gate and reused as
the slate's entry for gamePk 824256; the other 14 captured in one pass.

Note: this slate is the same 15 team-matchups as the v3 slate (2026-06-26) -- the next game in each series. It is an
independent capture (different starters, different market), kept in its own directional-contrast regime.

## 3. Cohort run manifest (15 runs, all 2026-06-28, RunType sports.matchup.analysis, artifact v3)

`lean` = persisted `AgentRun.LeanSide`. All runs `completed`, `mlb_statsapi` provider, `ExclusionReason IS NULL`,
tenantKey 1, posture `monitor`, artifact `sports_decision_artifact_v3`, evidence richness 2 (starting_pitching +
market both grounded). Full ids + market block in the machine manifest (sec 7).

| run | matchup (away @ home) | gamePk | lean | conf | rich | dirConsistency |
|---|---|---|---|---|---|---|
| 3b37433e | Houston Astros @ Detroit Tigers | 824256 | away | 0.80 | 2 | Consistent |
| 4337433e | Cincinnati Reds @ Pittsburgh Pirates | 823362 | home | 0.75 | 2 | Consistent |
| 4537433e | Texas Rangers @ Toronto Blue Jays | 822795 | home | 0.75 | 2 | Consistent |
| 4b37433e | Seattle Mariners @ Cleveland Guardians | 824422 | home | 0.75 | 2 | Consistent |
| 4c37433e | Arizona Diamondbacks @ Tampa Bay Rays | 822959 | home | 0.80 | 2 | Consistent |
| 4f37433e | Philadelphia Phillies @ New York Mets | 823608 | home | 0.75 | 2 | **PotentialMismatch** |
| 5237433e | Colorado Rockies @ Minnesota Twins | 823686 | home | 0.75 | 2 | Consistent |
| 5437433e | Kansas City Royals @ Chicago White Sox | 824580 | home | 0.75 | 2 | Consistent |
| 5737433e | Chicago Cubs @ Milwaukee Brewers | 823769 | home | 0.80 | 2 | Consistent |
| 5d37433e | Miami Marlins @ St. Louis Cardinals | 823037 | home | 0.75 | 2 | Consistent |
| 6237433e | Athletics @ Los Angeles Angels | 824011 | away | 0.75 | 2 | Consistent |
| 6437433e | Atlanta Braves @ San Francisco Giants | 823204 | away | 0.75 | 2 | Consistent |
| 6a37433e | Los Angeles Dodgers @ San Diego Padres | 823281 | home | 0.75 | 2 | Consistent |
| 7137433e | New York Yankees @ Boston Red Sox | 824744 | home | 0.80 | 2 | Consistent |
| 3c37433e | Washington Nationals @ Baltimore Orioles | 824821 | home | 0.75 | 2 | Consistent |

## 4. Cohort summary

- **Lean distribution: home 12, away 3, null 0.** (v3 was 10/4/1.) Confidence clusters 0.75-0.80 (10 at 0.75, 4 at
  0.80, plus the mismatch at 0.75) -- low variance, consistent with [[calibration-assessment-v3]] ("confidence is not
  predictive").
- **No thin runs this slate.** Every game had a probable starter published at capture, so all 15 are evidence
  richness 2 (starting_pitching + market grounded). This differs from v3 (12 enriched / 3 thin) -- v4 has no
  enriched-vs-thin depth contrast within the slate, but the directional contrast (home/away leans, market agreement
  and disagreement) is present.
- **Advertised strength:** all richness-2 -> standard deriver band `High` (the thin-evidence humility cap fires only
  on richness-1 runs, of which there are none here).

## 5. Integrity scan (fresh cohort) -- 1 documented blocker

Ran the lean/artifact direction-consistency scan over all 15 runs (read-time `artifactDirectionConsistency`, the
Decision Encoding Integrity guard).

- persisted `LeanSide` == artifact `LeanSide`: **15/15 match** (verified by DB read of `AgentRuns.LeanSide` vs
  `JSON_VALUE(OutputJson,'$.LeanSide')`).
- direction consistency: **14 Consistent, 1 PotentialMismatch.**
- game identity present: **15/15** (`mlb_statsapi` + gamePk, persisted + artifact agree).
- duplicate same-game runs: **none** (15 distinct gamePks).
- market baseline linkage: **15/15** valid.
- **0 outcomes / 0 evaluations written at capture** (DB-verified).

**The PotentialMismatch run -- 4f37433e, Philadelphia Phillies @ New York Mets (gamePk 823608):**

- structured persisted `LeanSide = home` (Mets), but the artifact prose says *"Slight lean toward Phillies based on
  starting pitching advantage and market support"* (Phillies = away). Guard verdict:
  `status=PotentialMismatch`, `reason=prose_points_to_opposite_side_in_lean`, `structuredLeanSide=home`,
  `detectedProseSides=[away]`.
- This is the same lean-vs-prose contradiction class as the historically-remediated 077A / BDDE423E / 01AA433E runs
  ([[mismatch-remediation-v1]]) and the still-unsettled 722D433E.
- **Disposition (per the 722D precedent, [[mismatch-remediation-v1]] sec 6): LEFT AS-IS, FLAGGED. Not excluded, not
  mutated.** Rationale: the run is unsettled and has produced no verdict; the live Settlement Direction Integrity
  guard (`AgentRunsController.RecordOutcome` -> `SettlementDirectionIntegrity`) will **422-refuse** to grade it while
  the contradiction stands, so it can never silently enter a verdict. Exclusion-as-`invalid` is applied only to runs
  that were *settled before the guard existed* (BDDE/01AA), to remove a wrongly-recorded verdict from the
  denominator; that does not apply to an unsettled run.
- Root cause is the known analyzer lean/prose disagreement; the standing fix is **Analyzer Lean-Agreement Hardening
  v1** (degrade `lean_side` to null at generation when it disagrees with the prose). This run is the live evidence
  that the contradiction is still produced at the source.

## 6. Market baseline v3 (de-vigged consensus moneyline)

Source: the-odds-api.com v4 `baseball_mlb` h2h, us region, american odds, captured pregame 2026-06-28. Per-book
two-way de-vig (normalize home+away implied to 1.0), consensus = mean across books (same method as
[[directional-contrast-cohort-v3-market-baseline-v2|v2]]). All 15 market-backed, 9 books each.

| run | matchup | devig H | devig A | favorite | favProb | DAI lean | agree? |
|---|---|---|---|---|---|---|---|
| 3c37433e | Washington Nationals @ Baltimore Orioles | 0.633 | 0.367 | home (Orioles) | 0.633 | home | yes |
| 4337433e | Cincinnati Reds @ Pittsburgh Pirates | 0.551 | 0.449 | home (Pirates) | 0.551 | home | yes |
| 4537433e | Texas Rangers @ Toronto Blue Jays | 0.546 | 0.454 | home (Blue Jays) | 0.546 | home | yes |
| 4c37433e | Arizona Diamondbacks @ Tampa Bay Rays | 0.630 | 0.370 | home (Rays) | 0.630 | home | yes |
| 4b37433e | Seattle Mariners @ Cleveland Guardians | 0.499 | 0.501 | away (Mariners) | 0.501 | home | **NO** |
| 3b37433e | Houston Astros @ Detroit Tigers | 0.456 | 0.544 | away (Astros) | 0.544 | away | yes |
| 4f37433e | Philadelphia Phillies @ New York Mets | 0.427 | 0.573 | away (Phillies) | 0.573 | home* | **NO*** |
| 5237433e | Colorado Rockies @ Minnesota Twins | 0.596 | 0.404 | home (Twins) | 0.596 | home | yes |
| 5737433e | Chicago Cubs @ Milwaukee Brewers | 0.635 | 0.366 | home (Brewers) | 0.635 | home | yes |
| 5437433e | Kansas City Royals @ Chicago White Sox | 0.555 | 0.445 | home (White Sox) | 0.555 | home | yes |
| 5d37433e | Miami Marlins @ St. Louis Cardinals | 0.542 | 0.458 | home (Cardinals) | 0.542 | home | yes |
| 6237433e | Athletics @ Los Angeles Angels | 0.476 | 0.524 | away (Athletics) | 0.524 | away | yes |
| 6437433e | Atlanta Braves @ San Francisco Giants | 0.409 | 0.591 | away (Braves) | 0.591 | away | yes |
| 6a37433e | Los Angeles Dodgers @ San Diego Padres | 0.440 | 0.560 | away (Dodgers) | 0.560 | home | **NO** |
| 7137433e | New York Yankees @ Boston Red Sox | 0.517 | 0.483 | home (Red Sox) | 0.517 | home | yes |

- **Market favorite distribution: home 9 / away 6.** All low-conviction (favProb 0.50-0.64).
- **DAI lean vs market favorite: 12 agree / 3 disagree** (on the persisted structured lean).
- `*` **Phillies@Mets is the PotentialMismatch run.** Its *structured* lean (home/Mets) disagrees with the market,
  but its *prose* leaned Phillies (away) -- which would AGREE with the market. The "disagreement" is therefore an
  artifact of the encoding contradiction, not a genuine contrarian read. **Clean disagreements this slate = 2**
  (Mariners@Guardians, Dodgers@Padres) + 1 mismatch-confounded.

## 7. Artifact traceability + machine manifest

Machine-readable manifest (full AgentRunIds, per-run market block + de-vig, integrity flags):
`04 Products/sports-v1/calibration/artifacts/20260628-directional-contrast-cohort-v4-manifest.json`.

Per-run verified: (a) DB row in `dbo.AgentRuns` (persisted LeanSide, ExternalGameId, SourceProvider, ExclusionReason
NULL); (b) artifact reads 200 via `GET /api/agent-runs/{id}/artifact`; (c) artifact `externalGameId` == StatsAPI
gamePk; (d) persisted `LeanSide` == artifact `LeanSide` (15/15); (e) market baseline attached by gamePk + odds event.
The manifest is sufficient for a settlement agent to reconcile each Consistent run by gamePk -> StatsAPI Final ->
`POST /api/agent-runs/{id}/outcome` (graded on persisted LeanSide). The PotentialMismatch run will 422 until remediated.

## 8. Settlement readiness and what remains before Calibration Assessment v4

- **Cohort is settlement-ready (UNSETTLED now).** 14 Consistent runs are gradable; the 1 PotentialMismatch will be
  guard-refused. Games settle ~2026-06-29T04:00Z+; reconcile 2026-06-29.
- **NOT settled this slice.** No outcomes/evaluations written; no scores guessed; corpus settled totals unchanged.
- **Cohort size adequacy: still pool-before-conclude.** With v3 now settled (74 valid), this is the 2nd
  directional-contrast market-baseline slate. Per [[calibration-assessment-v3]], pool >=3 settled slates before any
  directional/market conclusion; keep the directional-contrast regime separate from the 180x/190x/200x regimes.
- **Denominator doctrine unchanged:** settled AND `ExclusionReason IS NULL`. **Do NOT tune.**

## 9. No tuning / no change (held)

No prompt, confidence, source-depth, advertised-strength, buyer-copy, Tool Gateway, settlement, reconciliation,
calibration-logic, parser, posture, model, or Cognitive Factory change. Only evidence was captured: 15 new agent-run
rows + an external market baseline + an artifact trace manifest. Nothing was optimized. The one PotentialMismatch run
was left exactly as generated (flagged, not mutated).
