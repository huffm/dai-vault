# Directional-Contrast Cohort Settlement-Completion Pass v1

**date:** 2026-06-24
**status:** COMPLETE -- 9 of 10 cohort runs reconciled (1 postponed), plus 3 of 4 fresh-batch runs
(1 excluded). 12 outcomes posted against authoritative MLB StatsAPI finals. Corpus 35/35 -> 47/47.
**classification:** operational settlement slice. No code, model, prompt, schema, confidence, advertised-strength,
or reconciliation-logic change. Database writes only via the existing `POST /outcome` + `POST /exclude` endpoints.
**slice runner:** dai-slice-runner + dai-skill-router gate + superpowers:verification-before-completion.

**Anchor:** Grade only Final games, only on the persisted denormalized `AgentRun.LeanSide`. Outcomes come from
MLB StatsAPI (`schedule?sportId=1&hydrate=linescore`); the harness never guesses a score.

## 1. Question this slice answers

Can the Directional-Contrast cohort (deliberately balanced across home/away leans) show that DAI has directional
discrimination beyond the historical ~63% home-win base rate? **Answer: no -- not on this cohort.** The settled
cohort scored 2/7 decided (28.6%), below the base rate, with away leans 0/2 and home leans 2/5. Directional
discrimination remains UNPROVEN; n is small and this is one road-heavy slate, so this is not evidence the system is
worse than chance -- only that no edge was demonstrated.

## 2. Environment

- Docker Desktop started this slice; `devcore-sql` container started (Up :1433).
- DevCore.Api `:5007` health 200, run from HEAD `1a7c84c` (current main). FastAPI not needed (no generation).
- MLB StatsAPI reachable (HTTP 200); authoritative finals retrieved live for every reconciled gamePk.
- Repos at slice start: dai main `1a7c84c`, dai-vault main `6e8c1b2`, both synced with origin/main.

## 3. Readiness re-verification (live; prior audit not trusted)

Discrepancies found vs the 2026-06-24 readiness audit:

- **Reconciled count in the recent-25 window:** audit reported 6; live showed only 2 (`02aa433e`, `01aa433e`,
  both `correct`). The other settled runs had aged out of the recent-25 window. Whole-corpus pre-state was the
  expected 35/35 (verified by `sqlcmd`), so this is a windowing artifact, not a corpus discrepancy.
- **Duplicate landscape larger than audited.** The audit named "6 Yankees@Tigers dupes." Live, there were 6
  Yankees@Tigers runs total, one of which (`a6b5433e`) is the legitimate cohort member 200013 -> only **5** true
  duplicates. Live inspection also found duplicates the audit missed: a **Guardians@WhiteSox** dupe (`e5b5433e`,
  gamePk 824585) and a **Cubs@Mets** dupe (`d8b5433e`, gamePk 823613).
- **Cohort game 200018 (Cubs @ Mets, gamePk 823613) is POSTPONED**, not Final -- rescheduled to 2026-06-24. So
  only 9 of the 10 cohort runs were settle-eligible this pass.

All cohort gamePks and persisted LeanSides were confirmed live against the artifact endpoint and matched the
capture manifest. Run 722d433e persisted `LeanSide=home` (the read projection is contained to null on the
documented mismatch) -- confirming why it is excluded from calibration.

## 4. Exclusion pass (8 runs)

Via `POST /api/agent-runs/{id}/exclude` (tenant+user scoped; UserKey 1):

| run | matchup | gamePk | reason | result |
|---|---|---|---|---|
| d2b5433e | Yankees @ Tigers | 824261 | diagnostic | excluded |
| dab5433e | Yankees @ Tigers | 824261 | diagnostic | excluded |
| dbb5433e | Yankees @ Tigers | 824261 | diagnostic | excluded |
| deb5433e | Yankees @ Tigers | 824261 | diagnostic | excluded |
| e2b5433e | Yankees @ Tigers | 824261 | diagnostic | excluded |
| e5b5433e | Guardians @ White Sox | 824585 | diagnostic | excluded |
| d8b5433e | Cubs @ Mets | 823613 | diagnostic | excluded |
| 722d433e | Orioles @ Angels | 824017 | excluded | excluded |

The 7 `diagnostic` runs are near-close-capture duplicate reruns superseded by the identity-bearing cohort runs;
`722d433e` is excluded for the documented LeanSide/prose contradiction (Lean Direction Consistency Review v1). The
cohort members `a6b5433e` (Yankees@Tigers), `c7b5433e` (Guardians@WhiteSox), and `beb5433e` (Cubs@Mets) were kept
active. Excluded-run total 3 -> 11 (diagnostic 7, excluded 1, superseded 3 pre-existing).

## 5. Settlement results (12 outcomes posted)

All scores from MLB StatsAPI (`source=statsapi`, `sourceRef=statsapi:<gamePk>`); evaluation is the platform's
`RunEvaluator.Evaluate(persisted LeanSide, outcomeStatus)`.

### Directional-Contrast cohort (9 of 10; 200018 postponed)

| run | key | matchup | lean | conf | advStr | final (h-a) | outcome | eval |
|---|---|---|---|---|---|---|---|---|
| a6b5433e | 200013 | Yankees @ Tigers | away | 0.75 | High | 5-3 | home_win | incorrect |
| abb5433e | 200014 | Royals @ Rays | home | 0.75 | High | 1-2 | away_win | incorrect |
| b2b5433e | 200015 | Rangers @ Marlins | home | 0.75 | High | 3-4 | away_win | incorrect |
| b8b5433e | 200016 | Phillies @ Nationals | null | 0.45 | Medium | 4-1 | home_win | inconclusive |
| b9b5433e | 200017 | Astros @ Blue Jays | home | 0.75 | High | 4-2 | home_win | correct |
| beb5433e | 200018 | Cubs @ Mets | away | 0.75 | High | -- | POSTPONED | pending |
| c0b5433e | 200019 | Brewers @ Reds | null | 0.54 | Medium | 1-2 | away_win | inconclusive |
| c7b5433e | 200020 | Guardians @ White Sox | away | 0.75 | High | 6-5 | home_win | incorrect |
| cab5433e | 200021 | Dodgers @ Twins | home | 0.75 | High | 1-2 | away_win | incorrect |
| d1b5433e | 200022 | Diamondbacks @ Cardinals | home | 0.75 | High | 3-2 | home_win | correct |

Cohort settled: correct 2, incorrect 5, inconclusive 2 (9 total). Decided 7 -> **2/7 (28.6%)**.

### Fresh batch 2026-06-23 (3 of 4; 722d433e excluded)

| run | matchup | lean | conf | advStr | final (h-a) | outcome | eval |
|---|---|---|---|---|---|---|---|
| 6a2d433e | Mariners @ Pirates | away | 0.75 | High | 2-3 | away_win | correct |
| 702d433e | Red Sox @ Rockies | away | 0.80 | High | 2-5 | away_win | correct |
| 752d433e | Braves @ Padres | null | 0.45 | Medium | 7-6 | home_win | inconclusive |

Fresh settled: correct 2, inconclusive 1 (3 total).

## 6. Updated corpus totals (verified by sqlcmd, pre and post)

| metric | before | after | delta |
|---|---|---|---|
| outcomes | 35 | 47 | +12 |
| evaluations | 35 | 47 | +12 |
| correct | 20 | 24 | +4 |
| incorrect | 11 | 16 | +5 |
| inconclusive | 4 | 7 | +3 |
| decided | 31 | 40 | +9 |
| hit rate | 64.5% | **60.0%** | -4.5pp |

## 7. Directional-Contrast analysis (report only what the data supports)

- gradeable (non-null) leans settled: 7 (5 home, 2 away); plus 2 null (inconclusive) and 1 postponed.
- **home leans: 2/5 (40%)** -- abb/b2/cab incorrect, b9b/d1b correct.
- **away leans: 0/2 (0%)** -- a6b, c7b both incorrect.
- This was a road-heavy slate: 5 of the 7 decided games were away wins (Royals, Rangers, Brewers, Dodgers won
  on the road; the two away-lean picks Yankees/Guardians both lost). The home-lean default underperformed because
  road teams won; the away leans also missed.
- Confidence was flat 0.75 on every directional run, so it cannot discriminate WITHIN the cohort by construction.
  The 2/7 decided result means 0.75 was over-confident on this slate; it is NOT a basis to raise confidence.
- **Do not over-interpret:** n=7 decided is small and single-slate. This neither proves nor disproves an edge; it
  fails to DEMONSTRATE one, and on this slate the directional bet lost to the base rate.

## 8. Calibration assessment

1. **Still stalled? YES.** The one deliberately balanced cohort did not show directional discrimination; the
   corpus stays ~ home-base-rate (now 60.0%).
2. **Did the cohort move confidence? No** -- confidence is flat 0.75 across directional runs and cannot
   discriminate within-cohort; outcomes show it is over-stated on identity-only grounding, not under-stated.
3. **Still resemble home-base-rate behavior? YES** -- 60.0% decided, with the balanced cohort underperforming it.
4. **Another cohort needed? YES** -- one more balanced cohort (more away-lean games, mixed-winner slates) before
   any directional conclusion. Settle 200018 (Cubs@Mets) once it is replayed. No confidence tuning yet.
5. **Reconciled-Outcome Calibration Read v2 warranted? YES** -- the corpus grew 35 -> 47 with the first balanced
   directional data; a v2 read should consume it, keeping the directional cohort as its own regime (never pooled
   with the slate-confounded 180x/190x cohorts).

## 9. Verification

- Endpoints used: `GET /recent`, `GET /{id}/artifact`, `GET /{id}/evaluation`, `POST /{id}/exclude`,
  `POST /{id}/outcome` on DevCore.Api `:5007`.
- Outcome source: MLB StatsAPI schedule + linescore, queried live per gamePk; no score guessed or fabricated.
- Exclusions verified: 8 `exclude` calls returned 200; post-state `ExclusionReason` counts diagnostic 7 /
  excluded 1 (+ 3 pre-existing superseded).
- Settlement verified: 12 `outcome` posts returned 201; each `evaluation` read back the expected status; post-state
  `sqlcmd` totals 47/47 with 24/16/7, exactly +12 over the 35/35 pre-state.
- Claims verified: pre-state 35/35; cohort identities + leans; 9 finals + 1 postponed + 3 fresh finals; corpus
  47/47; cohort 2/7.
- Claims not verified: nothing material -- all figures are live DB reads this slice. (200018 outcome genuinely
  pending: game postponed.)

## 10. What did not change

No application source, no model/prompt/FastAPI, no schema/migration, no confidence/lean/posture logic, no
advertised-strength derivation, no reconciliation-matcher logic, no buyer copy, no Tool Gateway/source/probe/
billing/auth/tenant. Operational data only: 12 outcome+evaluation rows, 8 exclusion flags.

## 11. Next recommended slice

**Reconciled-Outcome Calibration Read v2** -- consume the 47-run corpus, keep the directional cohort as its own
regime, and report where the settled misses concentrate now that one balanced directional slate exists. Parallel:
a second Directional-Contrast Cohort Capture (more away leans, mixed-winner slates) to grow n; and settle 200018
(Cubs@Mets) after its replay. No confidence/prompt/posture tuning until v2 shows a discrimination signal.
