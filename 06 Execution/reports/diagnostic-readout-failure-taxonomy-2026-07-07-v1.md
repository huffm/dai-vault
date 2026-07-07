---
title: "Diagnostic Readout / Failure Taxonomy v1 -- market-attribution audit (2026-07-07)"
type: "report"
date: "2026-07-07"
status: "COMPLETE -- H1 SYSTEMIC: 6/6 divergences accidental; capture pause recommended until guard exists"
project: "DAI"
slice: "Diagnostic Readout / Failure Taxonomy v1"
related:
  - "06 Execution/reports/backed-depth-prediction-failure-analysis-2026-07-07-v1.md"
  - "06 Execution/reports/backed-depth-capture-settlement-attempt-handoff-2026-07-07-v2.md"
  - "06 Execution/reports/gate4-evidence-sufficiency-projection-2026-07-07-v1.md"
  - "06 Execution/reports/evidence-acquisition-cadence-proposal-2026-07-07-v1.md"
---

# diagnostic readout / failure taxonomy v1 -- market-attribution audit

## 1. purpose

diagnostic report only. audits the backed_depth artifact corpus for
staged-market-vs-prose consistency, splits the divergence ledger into deliberate /
accidental / unclear, and recommends whether capture cadence stays paused. it authorizes
no code, prompt, gate, tuning, buyer, or capture changes. gate-4 persisted counts are
untouched -- this report changes interpretation and reporting discipline only.

## 2. executive summary

**H1 is systemic, and broader than hypothesized.**

- **all six persisted marketAgreement=false rows in the corpus are ACCIDENTAL
  divergences (6/6; zero deliberate, zero unclear).** on every one, staged decision-time
  market data said the consensus opposed DAI's lean, and the artifact prose claimed the
  market SUPPORTED the lean. this includes both divergences that settled correct
  (824662, 824743) -- the 2/5 settled-disagreement record was produced by the same
  attribution defect as the 3 losses.
- **the defect is not confined to disagreement rows.** two agree rows misattribute the
  market too (824012 inverts it AGAINST dai's own lean; 824175 asserts both directions),
  and the no-lean canary 822882 inverts it as well. 9 of 38 artifacts (~24%) misstate
  the staged market in prose.
- **on every row where the staged market opposed the artifact's narrative direction
  (7 of 7: six divergences + the no-lean canary), the prose flipped the market to
  agree.** the market-rubber-stamp pattern is confirmed as one-directional conformity,
  with occasional random error on agree rows (2 of 31).
- consequence: the persisted marketAgreement field (computed from staged data) is the
  ONLY trustworthy market-relationship signal. artifact prose market language is
  unreliable and -- because the artifact is the buyer-facing product unit -- this is a
  product-quality defect, not just a calibration nuisance.
- **operational recommendation: PAUSE capture cadence until Market Attribution Fidelity
  Guard v1 exists** (section 9). settlement of the in-flight 2026-07-07 cohort still
  proceeds when finals post -- the rows are valid calibration evidence; only the edge
  interpretation changes.

## 3. corpus and method

- corpus: all 36 rows in live /rows with selectedDataRegime or observedDataRegime =
  starter_enriched_market_backed_depth (slates 06-29 x8, 06-30 x8, 07-03 x8 incl the
  no-lean canary, 07-06 x6, 07-07 x6), PLUS the two non-backed_depth settled
  disagreement rows (824500 06-24, 824743 06-26) so the ledger covers every persisted
  marketAgreement=false row. total 38 artifacts.
- method: for each run, GET /api/agent-runs/{id}/artifact (read-only); extract staged
  market evidence (sourceDepth market_odds detail: consensus side, median home win
  prob, book count/disagreement; persisted marketConsensusSide as the decision-time
  authority) and every market/consensus/moneyline sentence from summary, discern
  (contrast/weigh), counterCase, whatWouldChangeTheRead, and lean. prose market side
  resolved by team-name matching against homeTeamRef/awayTeamRef, manually verified on
  all disagreement rows and all ambiguous rows. full extracted text archived in the
  session digest (regenerable from live endpoints).
- one extraction caveat: 823529's artifact-side market detail lacks the consensus token
  (older artifact format); its persisted marketConsensusSide=away (decision-time
  snapshot join) is used as the staged authority. classification confidence unchanged.

## 4. taxonomy

1. **market-aligned prose** -- staged side and prose describe the same market preference.
2. **accidental divergence** -- staged/persisted side opposes DAI lean; prose claims or
   implies the market supports the lean. (a market-attribution failure, NOT candidate
   edge.)
3. **deliberate divergence** -- staged side opposes DAI lean; prose correctly
   acknowledges the market is against the read and explains the lean anyway. (the only
   class eligible for candidate-edge interpretation.)
4. **unclear divergence** -- staged data and prose cannot be compared safely.
5. **market-rubber-stamp pattern** -- prose describes the market as supporting the lean
   regardless of staged side (corpus-level pattern, assessed across rows).
6. **source/prose mismatch** -- general class: prose misrepresents staged source data,
   whether or not a divergence results (includes agree-row inversions and
   self-contradictions).

## 5. classification table (one row per artifact; 38 rows)

prose side = market side named in market sentences (manually verified where ambiguous).
class: A = market-aligned prose, AD = accidental divergence, SPM = source/prose mismatch.

| gamePk | slate | run | lean | staged mkt | agree | prose mkt | class | settled | eval |
|---|---|---|---|---|---|---|---|---|---|
| 824500 | 06-24 | fe79433e | home | away | false | home ("consensus favoring the Reds") | **AD** | away_win | incorrect |
| 824743 | 06-26 | 7704433e | home | away | false | home ("market shows a slight favor towards the Red Sox") | **AD** | home_win | correct |
| 822792 | 06-29 | 37bd433e | home | home | true | home | A | home_win | correct |
| 823444 | 06-29 | 33bd433e | away | away | true | away (Pirates; notes book disagreement) | A | away_win | correct |
| 823529 | 06-29 | 36bd433e | home | away* | false | home ("broad consensus favoring the Yankees") | **AD** | away_win | incorrect |
| 823771 | 06-29 | 48bd433e | away | away | true | away | A | home_win | incorrect |
| 824420 | 06-29 | 42bd433e | away | away | true | away | A | away_win | correct |
| 824662 | 06-29 | 4cbd433e | home | away | false | home ("slight favor towards the Cubs" -- Cubs are HOME) | **AD** | home_win | correct |
| 824741 | 06-29 | 3dbd433e | home | home | true | home | A | home_win | correct |
| 824822 | 06-29 | 2dbd433e | away | away | true | away (White Sox; notes book disagreement) | A | away_win | correct |
| 822793 | 06-30 | de40433e | home | home | true | home (Blue Jays 54%) | A | away_win | incorrect |
| 823122 | 06-30 | d040433e | home | home | true | home (Mariners) | A | home_win | correct |
| 823528 | 06-30 | db40433e | home | home | true | home | A | away_win | incorrect |
| 824096 | 06-30 | c440433e | away | away | true | away | A | away_win | correct |
| 824175 | 06-30 | ca40433e | away | away | true | BOTH ("slightly favors the Astros ... slightly favors the Twins") | **SPM** (self-contradiction) | home_win | incorrect |
| 824661 | 06-30 | d640433e | home | home | true | home | A | home_win | correct |
| 824907 | 06-30 | c240433e | home | home | true | home | A | away_win | incorrect |
| 824984 | 06-30 | cf40433e | away | away | true | away | A | away_win | correct |
| 822882 | 07-03 | c849433e | none | away | n/a | home ("strong support for the Texas Rangers") | **SPM** (no-lean row) | away_win | inconclusive |
| 823118 | 07-03 | cb49433e | home | home | true | home | A | home_win | correct |
| 824012 | 07-03 | dc49433e | away | away | true | home ("consensus favors the Angels on the moneyline") | **SPM** (inverted vs own lean) | away_win | correct |
| 824092 | 07-03 | d949433e | away | away | true | away | A | away_win | correct |
| 824171 | 07-03 | d049433e | home | home | true | home (Astros 52%) | A | home_win | correct |
| 824415 | 07-03 | cd49433e | home | home | true | home | A | away_win | incorrect |
| 824903 | 07-03 | d649433e | home | home | true | home | A | home_win | correct |
| 825063 | 07-03 | dd49433e | away | away | true | away | A | home_win | incorrect |
| 822712 | 07-06 | ad31433e | home | home | true | home | A | home_win | correct |
| 822958 | 07-06 | ac31433e | home | home | true | home | A | away_win | incorrect |
| 823036 | 07-06 | b431433e | home | away | false | home ("slight preference for the Cardinals") | **AD** | away_win | incorrect |
| 823205 | 07-06 | b731433e | away | away | true | away (Blue Jays) | A | home_win | incorrect |
| 823282 | 07-06 | b631433e | home | home | true | home | A | away_win | incorrect |
| 824900 | 07-06 | b331433e | home | home | true | home | A | away_win | incorrect |
| 822713 | 07-07 | aa2c433e | home | home | true | home | A | unsettled | -- |
| 822956 | 07-07 | a92c433e | home | home | true | home (note: invents "-1.5" spread language on an h2h-only staged market) | A | unsettled | -- |
| 823280 | 07-07 | ac2c433e | home | home | true | home | A | unsettled | -- |
| 823687 | 07-07 | 9e2c433e | home | home | true | home (Twins 54%) | A | unsettled | -- |
| 824579 | 07-07 | b32c433e | away | away | true | away (Red Sox) | A | unsettled | -- |
| 824820 | 07-07 | a32c433e | home | away | false | home ("slight favor towards the Orioles") | **AD** | unsettled | -- |

\* 823529: artifact staged detail lacks consensus token; persisted decision-time
marketConsensusSide=away used (see section 3 caveat).

**summary counts:** 38 checked (31 agree / 6 disagree / 1 no-lean). accidental
divergences **6**; deliberate **0**; unclear **0**. source/prose mismatch total **9**
(6 AD + 824012 + 824175 + 822882) = ~24% of corpus. market-rubber-stamp: confirmed --
7 of 7 rows where staged opposed the narrative direction were flipped in prose;
agree-row prose correct 29 of 31.

## 6. candidate-edge ledger split

| gamePk | matchup | cohort | settled | lean | staged mkt | agree | class | result | counts as candidate edge? | notes |
|---|---|---|---|---|---|---|---|---|---|---|
| 824500 | MIL@CIN | 06-24 | yes | home | away | false | accidental | incorrect | **NO** | prose claimed Reds market support; staged away |
| 824743 | NYY@BOS | 06-26 | yes | home | away | false | accidental | correct | **NO** | won, but via the same attribution defect |
| 823529 | DET@NYY | 06-29 | yes | home | away | false | accidental | incorrect | **NO** | prose claimed Yankees market support |
| 824662 | SD@CHC | 06-29 | yes | home | away | false | accidental | correct | **NO** | won; Cubs are home; prose claimed Cubs market support vs staged away Padres |
| 823036 | MIL@STL | 07-06 | yes | home | away | false | accidental | incorrect | **NO** | failure-analysis H1 index case |
| 824820 | CHC@BAL | 07-07 | no | home | away | false | accidental | pending | **NO** | classified pre-settlement (settlement resume handoff v2) |

- **zero rows currently qualify for candidate-edge interpretation.** the settled
  disagreement record (2 correct / 3 incorrect) still validly measures "when the
  pipeline's output lands opposite the market, is it right?" -- persisted fields are
  decision-time truth -- but it measures the byproduct of an attribution defect, not
  deliberate contrarian judgment, and must not be narrated as edge-seeking.
- this ledger split does NOT change gate-4 persisted counts (marketDisagreementN stays
  as computed by the live criterion). it changes interpretation and future reporting
  discipline: readouts must annotate divergence rows with the taxonomy class.

## 7. findings (direct answers)

1. **is divergence-targeted capture valid?** partially. it reliably produces
   market-covered rows and persisted-disagreement rows (both real gate-4 needs). it is
   NOT producing what it was named for -- deliberate contrarian reads -- because none
   exist yet.
2. **contrarian reads or attribution defects?** attribution defects: 6/6.
3. **isolated to marketAgreement=false rows?** no. 2 agree rows + the no-lean canary
   misattribute too (~24% of corpus overall). disagreement rows are where the defect is
   GUARANTEED to surface (7/7), agree rows mostly mask it.
4. **does prose systematically conform market language to the lean?** yes on every
   staged-opposed row (7/7). agree-row errors look more random (one inversion against
   the lean, one self-contradiction), so the general defect is "market attribution is
   unreliable," with conformity bias dominant.
5. **does the direction-integrity guard cover this?** no -- by design it compares
   structured lean to prose LEAN direction only. all 38 rows pass it. prose-vs-staged-
   market is an unguarded class.
6. **readout/reporting language changes (immediate):** persisted marketAgreement=false
   rows are called "market-opposed rows"; the taxonomy class must be stated per row;
   "candidate edge signal" is reserved for deliberate divergences (currently zero);
   accidental rows are "accidental divergence / market-attribution failure." the
   2026-07-07 settlement readout must carry the 824820 accidental annotation.
7. **pause until a guard exists?** yes -- recommendation in section 9.

## 8. guard/spec seed (docs-only; implementation is a future slice)

**Market Attribution Fidelity Guard v1** -- checks whether artifact prose faithfully
represents staged market data.

- inputs: staged marketConsensusSide, staged median home/away implied probabilities,
  staged market favorite side; artifact summary, discern market language, counter-case
  market language, whatWouldChangeTheRead market language, final lean explanation.
- outputs: PASS | FAIL_MARKET_ATTRIBUTION_MISMATCH | UNCLEAR_MARKET_ATTRIBUTION.
- behavior: FAIL when prose claims market support for the lean while the staged side is
  opposite; warn-or-fail when prose omits market opposition entirely on
  marketAgreement=false rows; on FAIL, mark the row's divergence class = accidental;
  does NOT block settlement by itself; DOES block candidate-edge interpretation and
  divergence-ledger credit. deterministic, evaluated from persisted artifact + staged
  fields (same fail-soft philosophy as the direction-integrity guard).
- placement question for the implementation slice: compose-time warning (artifact
  quality) vs settlement-time classification (ledger integrity) vs both. the corpus
  evidence (buyer-visible prose asserting false market claims) argues for both.

## 9. immediate operational recommendation

**pause capture cadence until Market Attribution Fidelity Guard v1 exists.** chosen over
"continue with manual H1 screen" because: (a) 6/6 divergences accidental -- the cadence's
divergence yield is currently 100% defect; (b) the defect contaminates buyer-visible
prose on ~24% of rows, so each capture morning also produces product-quality liabilities;
(c) a manual screen catches the ledger problem but not the artifact-quality problem; and
(d) the numeric gate needs (coverage +2, disagreement +5 per the projection) lose nothing
from a short pause -- the 07-07 cohort already in flight will deliver +6 covered rows and
+1 (accidental) disagreement row when settled. settlement work continues unaffected.

## 10. what this does not license

- no tuning
- no threshold edits
- no model replacement
- no prompt changes
- no buyer-facing accuracy, edge, or performance claims
- no gate edits
- no registry default-on change
- no new capture authorization
- no settlement authorization

## 11. recommended next slice

H1 is systemic -> **Market Attribution Fidelity Guard v1** (code slice, TDD,
approval-gated) is the next build. sequencing: (1) settle the 2026-07-07 cohort first
when finals post (~01:00 ET 07-08), with the readout carrying the accidental annotation
for 824820 and this taxonomy's language rules; (2) then the guard slice; (3) capture
cadence resumes only after the guard is live. also carry forward two audit byproducts:
gamePk 824662 has TWO active rows (2cde423e 06-28 unsettled + 4cbd433e 06-29 settled) --
a reconcile-precheck hazard to resolve before anyone settles that gamePk again; and
822956's prose invents "-1.5" spread language on an h2h-only staged market (same
fidelity family; the guard spec should consider market-TYPE fidelity too).
