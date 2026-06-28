# Cohort v3 Settlement v1 -- Directional-Contrast Cohort (2026-06-26 slate)

**date:** 2026-06-28
**status:** SETTLED. The 15-run Directional-Contrast Cohort v3 (captured 2026-06-26, all games now Final) was
reconciled against StatsAPI finals via the platform's own settlement endpoints. 15/15 outcomes recorded, server-graded.
No tuning; no code/prompt/confidence/model/buyer change; no Cognitive Factory activation. Settlement writes only.
**type:** Lane A data-quality settlement slice. dai-slice-runner + dai-skill-router gate + systematic verification +
verification-before-completion.
**anchor:** Cohort v3 was settlement-ready but unsettled ([[directional-contrast-cohort-v3-market-baseline-v2]]).
All games are final, identity-safe, and integrity-clean, so settling is the highest-value Gate-4 accumulation step.

See also: [[directional-contrast-cohort-v3-market-baseline-v2]] (the captured cohort), [[calibration-assessment-v3]]
(frozen baseline, 59-run -> 74 valid after this settlement), [[mismatch-remediation-v1]] (denominator doctrine),
[[system-state-synopsis-v1]].

## 1. Method (integrity-preserving)

Reconciled per `agentRunId` against StatsAPI finals -- NOT by collapsing gamePks. For each run:
`POST /api/agent-runs/{id}/outcome` with the StatsAPI Final score + derived `outcomeStatus` (home_win / away_win;
no draws in MLB), then `GET /api/agent-runs/{id}/evaluation` to read back the platform's verdict. The server grades
on the **persisted `AgentRun.LeanSide`** (the client never sends a lean), and the **Settlement Direction Integrity
guard** (422) would block any self-contradictory run before writing. Scores were taken from StatsAPI
(`schedule?sportId=1&date=2026-06-26`), never guessed. Only `Final` games with numeric scores were posted; all 15
were Final, 0 in progress, 0 postponed/cancelled/suspended.

## 2. Settlement results (server-graded, DB-verified)

| run | matchup (away @ home) | gamePk | final (H-A) | persisted lean | outcome | eval |
|---|---|---|---|---|---|---|
| 4d04433e | Houston Astros @ Detroit Tigers | 824255 | 8-0 | home | home_win | **correct** |
| 5104433e | Cincinnati Reds @ Pittsburgh Pirates | 823363 | 4-6 | home | away_win | incorrect |
| 5704433e | Washington Nationals @ Baltimore Orioles | 824824 | 3-1 | home | home_win | **correct** |
| 5c04433e | Texas Rangers @ Toronto Blue Jays | 822796 | 4-5 | away | away_win | **correct** |
| 6304433e | Seattle Mariners @ Cleveland Guardians | 824423 | 1-3 | home | away_win | incorrect |
| 6a04433e | Arizona Diamondbacks @ Tampa Bay Rays | 822962 | 6-1 | home | home_win | **correct** |
| 7104433e | Philadelphia Phillies @ New York Mets | 823610 | 1-2 | null | away_win | inconclusive |
| 7704433e | New York Yankees @ Boston Red Sox | 824743 | 6-1 | home | home_win | **correct** |
| 7c04433e | Kansas City Royals @ Chicago White Sox | 824582 | 22-1 | home | home_win | **correct** |
| 8304433e | Chicago Cubs @ Milwaukee Brewers | 823773 | 6-2 | home | home_win | **correct** |
| 8704433e | Colorado Rockies @ Minnesota Twins | 823688 | 9-8 | home | home_win | **correct** |
| 8c04433e | Miami Marlins @ St. Louis Cardinals | 823039 | 0-4 | home | away_win | incorrect |
| 8d04433e | Athletics @ Los Angeles Angels | 824014 | 3-9 | away | away_win | **correct** |
| 9404433e | Los Angeles Dodgers @ San Diego Padres | 823285 | 7-1 | away | home_win | incorrect |
| 9704433e | Atlanta Braves @ San Francisco Giants | 823206 | 1-3 | away | away_win | **correct** |

## 3. Settlement summary

- **Total cohort: 15. Settled: 15. Unsettled: 0. Excluded: 0.** (All Final, identity-safe; nothing ambiguous,
  postponed, duplicate, or identity-conflicted.)
- **Verdicts: 10 correct / 4 incorrect / 1 inconclusive.**
  - Decided (non-null lean): 14. **DAI directional accuracy on this slate: 10/14 = 71.4%.**
  - The 1 inconclusive is Phillies@Mets, where DAI abstained (null lean -- no starter at capture).
- **DAI vs market favorite (from the captured de-vig baseline):** the lone DAI-vs-market disagreement,
  **NY Yankees @ Boston Red Sox**, resolved in **DAI's favor** -- DAI leaned home (Red Sox), market favored
  Yankees; Red Sox won 6-1. Market favorite went 10-5 over the 15 games. **n=1 disagreement -- an observation to
  pool, not an edge.**
- **Integrity blockers: none.** No PotentialMismatch in this cohort (the capture scan was clean); no run was
  guard-refused; the null-lean run graded inconclusive (correct -- no side to grade).

## 4. Verification (commands + DB evidence)

- Settlement writes via existing endpoints only (`POST .../outcome`, `GET .../evaluation`). DevCore.Api built/run
  from HEAD `fee495d`; dev bypass -> tenantKey 1 (the cohort's tenant).
- **No duplicate writes:** a second settlement pass returned `409 Conflict` on all 15 (idempotent by `AgentRunKey`),
  verdicts unchanged.
- **DB cross-check (devcore-sql):** the 15 cohort runs (by gamePk) have exactly 15 outcomes + 15 evaluations
  (1 each), 0 excluded; eval breakdown 10 correct / 4 incorrect / 1 inconclusive; 0 runs with duplicate outcomes
  corpus-wide. Corpus-wide `outcomes_total == evaluations_total == 76` (no orphans).
- **Denominator after settlement:** valid-calibration set (settled AND `ExclusionReason IS NULL`) = **74**
  (was 59; +15). Doctrine unchanged ([[mismatch-remediation-v1]] sec 5).
- No confidence/prompt/model/posture/lean/buyer/tuning change. No Cognitive Factory activation.

## 5. What this does and does not unlock

- **Does:** grows the valid denominator to 74 and produces the first settled directional-contrast + market-baseline
  slate. This is the 1st of the >=3 settled directional-contrast slates needed before reading DAI vs market vs outcome.
- **Does NOT:** justify Calibration Assessment v4 yet. A single settled directional-contrast slate cannot clear
  Gate 4. Keep this regime separate from 180x/190x/200x. Pool >=3 settled directional-contrast slates first
  (Cohort v4, captured 2026-06-28, is the 2nd once settled). **Do NOT tune.**
