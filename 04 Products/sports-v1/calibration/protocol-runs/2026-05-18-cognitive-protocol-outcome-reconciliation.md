# Outcome Reconciliation -- 2026-05-18 Cognitive Protocol Calibration Batch

| field | value |
|---|---|
| generated | 2026-05-20 19:44 |
| mode | live |
| export | 2026-05-18-cognitive-protocol-run-export.csv |
| results input | 2026-05-18-game-results-input.csv |
| reconciled output | 2026-05-18-cognitive-protocol-run-export-reconciled.csv |
| runs in batch | 8 |
| completed | 8 |
| pending | 0 |
| outcomes posted | 8 |

## completed games

| matchup | final score (home-away) | actual winner | lean side | directional eval | cover | total | status |
|---|---|---|---|---|---|---|---|
| San Antonio Spurs at Oklahoma City Thunder | 115-122 | San Antonio Spurs | Oklahoma City Thunder | incorrect | not_recorded | not_recorded | reconciled |
| Cleveland Cavaliers at New York Knicks | 115-104 | New York Knicks | New York Knicks | correct | not_recorded | not_recorded | reconciled |
| Atlanta Braves at Miami Marlins | 12-0 | Miami Marlins | Atlanta Braves | incorrect | not_recorded | not_recorded | reconciled |
| Baltimore Orioles at Tampa Bay Rays | 16-6 | Tampa Bay Rays | none | inconclusive | not_recorded | not_recorded | reconciled |
| Cincinnati Reds at Philadelphia Phillies | 5-4 | Philadelphia Phillies | none | inconclusive | not_recorded | not_recorded | reconciled |
| Cleveland Guardians at Detroit Tigers | 2-8 | Cleveland Guardians | Cleveland Guardians | correct | not_recorded | not_recorded | reconciled |
| New York Mets at Washington Nationals | 7-16 | New York Mets | none | inconclusive | not_recorded | not_recorded | reconciled |
| Toronto Blue Jays at New York Yankees | 7-6 | New York Yankees | New York Yankees | correct | not_recorded | not_recorded | reconciled |

## pending games

None. Every game has a final result.

## directional evaluation

From the platform `AgentRunEvaluation` (computed from `AgentRun.LeanSide` vs the recorded winning side):

- correct: 3
- inconclusive: 3
- incorrect: 2

## method and caveats

- Outcomes are posted only for games marked `final` with numeric scores in the results input.
- Cover and total results are filled only when `closing_spread_home` / `closing_total` are supplied in the results input. Blank means not computed -- the harness never guesses a line or total.
- 3 of 8 runs have `lean_side = none` (signals were split); their platform directional evaluation will be `inconclusive`.
- The original export CSV is never modified. The reconciled CSV is harness output and is regenerated on each run.
- Confidence rules, the analyzer, FastAPI, and runtime scoring logic were not touched. The harness only calls `POST /api/agent-runs/{id}/outcome` and `GET /api/agent-runs/{id}/evaluation`.


