# Market-Aware MLB Miss Signal-Gap Review v1

**date:** 2026-06-18
**status:** HYPOTHESIS GENERATION - tentative, pending settlement of the remaining 6 runs.
**classification:** post-outcome research; docs-only. Not a source-priority decision, not calibration truth.

**Anchor:** New evidence regime, new calibration baseline.

## 0. Strict Framing (read first)

- **n=2 only.** Two reconciled market-aware moderate runs, both home leans that lost. Same direction. Not a sample.
- **Post-outcome research.** Leads were seeded after the games were Final. Postgame box facts are used to *test* the leads, never treated as pregame-known evidence.
- **Hypothesis generation, not calibration truth.** This produces candidate source-depth hypotheses to revisit later; it concludes nothing about model quality or thresholds.
- **No advisory/enforcement implication.** Observed PerceiveFulfillment stays neutral and read-only; nothing here promotes advisory or enforcement.
- **No source priority is final** until the remaining 6 market-aware runs settle and are reconciled (target ~2026-06-19T02:00Z). Any ranking below is provisional.

## 1. Highest-Value Finding

The most useful finding is **not** "market failed." Both artifacts already *named the right risks*: `b0de423e` flagged "Pirates may have a stronger lineup against right-handed pitchers" in its `counterCase`, and `b4de423e` flagged "Orioles could exploit any weaknesses in Kirby's performance, but specific data is lacking." The leans rode market run-line support plus home-field advantage.

The gap is therefore **grounded depth, not model awareness**: `starting_pitching` is grounded at probable-starter *identity* only (names/handedness), with no starter quality/form, and there is no grounded team/lineup context. The model surfaced the correct uncertainty but had nothing grounded to weigh against the market read.

## 2. Misses Under Review

| run | gamePk | result | lean (structured) | eval |
|---|---|---|---|---|
| b0de423e | 824992 | Pirates 12 @ Athletics 4 | home | incorrect |
| b4de423e | 823127 | Orioles 5 @ Mariners 3 | home | incorrect |

Both grounded `starting_pitching` + `market`, `SourceSufficiency=moderate`, observed `decision=0`/Fulfilled, evidence richness 2, confidence 0.75, posture monitor.

## 3. Postgame Verification (StatsAPI boxscore - public postgame source)

Used only to test the seeded leads. These are realized in-game facts, not pregame evidence.

**824992:**
- Aaron Civale (A's, home starter): 3.0 IP, 9 H, 6 R, 6 ER, 2 K - early blowup confirmed.
- Ryan O'Hearn (PIT): 5 AB, 3 H, 1 HR, **6 RBI** - extreme single-game outlier confirmed.
- Scott Barlow (A's pen): 0.1 IP, 4 R, 4 ER - bullpen non-stabilization confirmed.
- Braxton Ashcraft (PIT starter): 6.0 IP, 1 ER, 7 K - opposing starter strong.

**823127:**
- Kyle Bradish (BAL, away starter): 7.2 IP, 5 H, 1 ER, 2 BB, **12 K** - dominant outing confirmed.
- George Kirby (SEA, home starter): 6.0 IP, 3 ER, 5 K - competitive, a few decisive mistakes (partial).
- Mariners' 3 runs came entirely on two home runs (Canzone, Young) - thin offensive support consistent with the lead; broader pattern not verifiable from one box.
- Julio Rodriguez in-game exit / Tyler O'Neill defensive swing: not present in the box summary; in-game-only regardless of verification.

## 4. Lead Classification

Categories: **PRE** pregame-available | **IGP** in-game/postgame-only | **REP** already represented in current signals | **MSG** missing source group | **FNS** feasible next source | **DEF** defer/no action.

### Miss 1 - b0de423e / 824992

| lead | classification | source-group mapping / note |
|---|---|---|
| Civale return-from-injury status | PRE; MSG; FNS | starting_pitching (availability/quality); statsapi has IL/status |
| Civale 6 ER / 3 IP line | IGP | realized performance, not pregame-knowable |
| O'Hearn 6-RBI outlier | IGP; DEF | extreme variance; not a source target |
| Pirates recent offensive form | PRE; MSG; FNS | team_form |
| Bullpen meltdown sequence (Barlow) | IGP | realized |
| Bullpen readiness/recent usage | PRE; MSG; FNS | bullpen_availability (harder source) |
| lineup_injury / availability | PRE (known scratches/IL); MSG; FNS | statsapi lineup |
| starter quality beyond identity | REP-as-group, identity-only depth; MSG(depth); FNS | deepen existing signal |
| pregame hypotheses (L5) | PRE | the legitimately actionable subset |
| postgame variance (L6) | IGP; DEF | variance, not a source target |

Already represented / used: starter *identity* (both RHP), market run-line (home), home-field. counterCase named the lineup-vs-RHP risk but it was not grounded.

### Miss 2 - b4de423e / 823127

| lead | classification | source-group mapping / note |
|---|---|---|
| Bradish K / pitch-form profile | PRE; MSG; FNS | starting_pitching (quality) |
| Bradish 12-K outing | IGP | realized |
| Mariners known pregame absences | PRE; MSG; FNS | lineup_injury |
| Julio Rodriguez in-game exit | IGP; DEF | separate from pregame injury risk (per lead) |
| Kirby decisive mistakes | IGP | realized |
| Kirby recent form (pregame) | PRE; MSG; FNS | starting_pitching (quality) |
| team_form / run support | PRE; MSG; FNS | team_form |
| bullpen_availability | MSG; FNS | harder source |
| defensive / run-prevention variance | MSG (no existing taxonomy group); DEF | needs a definition first; realized swings were IGP |
| pregame hypotheses (L6) | PRE | actionable subset |
| postgame variance (L7: Julio exit, O'Neill robbed HR, 9th-inning rally) | IGP; DEF | variance, not a source target |

Already represented / used: starter *identity* (Kirby vs Bradish, named), market run-line (home), home-field. counterCase said "specific data is lacking" - the gap is grounding depth, not awareness.

## 5. Provisional Source-Depth Hypotheses (NOT a decision)

Ranked by cross-miss leverage x feasibility. Provisional; revisit only after the remaining 6 settle.

1. **starting_pitching quality/form** (beyond probable-starter identity). Appears in both misses (Civale, Bradish, Kirby). Enriches an already-grounded group -> lowest new-source risk; statsapi carries season stats + recent game logs. Caveat: this is *depth* on an existing band group - watch that it does not silently inflate `evidence_richness`/band mechanics.
2. **team_form / recent offensive production.** Both misses (Pirates damage; Mariners thin run support). Feasible via statsapi splits.
3. **lineup_injury / pregame availability.** Both misses cite it; feasible for known pregame scratches/IL. The decisive items (Julio in-game exit) are IGP and out of reach.
4. **bullpen_availability.** Miss 1 only (Barlow). Real value but the hardest free source (usage/availability not clean in a public feed). Medium feasibility.
5. **market confirmation (sharp_public / line_movement).** Both artifacts already flag these as missing/candidate follow-ups; `line_movement` is a `future_candidate`. A confirmation channel for the existing market read, not a new directional group. Lower priority than evidence breadth.

Deferred: defensive/run-prevention variance (no taxonomy group; would need a definition; realized swings were IGP).

## 6. Guardrails

- **No single missing signal would have flipped either lean.** Both leans rode market + home-field; the misses were dominated by postgame variance (O'Hearn 6 RBI; Civale 6 ER/3 IP; Barlow 4 ER/0.1 IP; Bradish 12 K). Pregame enrichment would at most widen the counter-case or lower confidence - not deterministically flip to away.
- The artifacts named the right risks; the gap is grounded depth (starter quality/form) and breadth (team_form, lineup_injury), not model awareness.
- n=2, same direction. Directional only. Confirm/extend against the 6 pending runs after settlement before any source decision.

## 7. Relationship to Prior Docs

- Tests the structural finding in `source-coverage-and-calibration-variance-plan-v1.md` (variance, not volume, is the bottleneck) against real misses, within the market-aware regime only.
- Does not reopen `market-aware-mlb-stage-0-capture-v1.md` or `reconcile-market-aware-mlb-moderate-cohort-v1.md`; this is post-outcome research layered on top.
- No advisory/enforcement implication for `perceive-fulfillment-vs-outcome-calibration-review-v1.md`; observed mode stays neutral.

## 8. Recommended Next Slice

Settlement-completion pass on the remaining 6 market-aware runs (after ~2026-06-19T02:00Z), then re-run this gap review across all 8 settled outcomes before treating any source-depth hypothesis as a priority. Source integration remains deferred and out of scope until then.
