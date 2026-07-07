---
title: "Backed-Depth Prediction Failure Analysis v1 (2026-07-06 cohort, 1/6)"
type: "report"
date: "2026-07-07"
status: "COMPLETE -- diagnostic only; authorizes no changes"
project: "DAI"
slice: "Backed-Depth Prediction Failure Analysis v1"
related:
  - "06 Execution/reports/backed-depth-divergence-settlement-handoff-2026-07-07-v1.md"
  - "06 Execution/reports/gate4-evidence-readout-backed-depth-divergence-2026-07-06-v1.md"
  - "06 Execution/reports/gate4-evidence-sufficiency-projection-2026-07-07-v1.md"
---

# backed-depth prediction failure analysis v1 (2026-07-06 cohort)

## 1. purpose

failure analysis of the settled 2026-07-06 backed_depth cohort (1 correct / 5 incorrect,
0.167). **this is a diagnostic, not a tuning authorization.** every claim below is
traceable to the six AgentRun artifacts (OutputJson via /artifact), /rows, the settlement
handoff, or the filled readout. sources: artifact JSONs pulled read-only 2026-07-07;
signal-diagnostics discipline per dai-signal-follow-up-diagnostics (signal vocabulary,
phase ownership, no invented staleness).

## 2. executive summary

1/6 was poor, and the divergence failed. the headline findings, in order of importance:

1. **the cohort's celebrated "first backed_depth divergence" (823036 MIL@STL) was an
   accident, not a contrarian read.** the staged market data said `consensus away, median
   home win prob 50%`; the artifact's summary and discern phase BOTH claim "the market
   shows a slight preference for the Cardinals" (home). the model believed the market
   agreed with its home lean. the persisted marketAgreement=false row is real, but the
   *reasoning* never knowingly disagreed with the market -- a **market-direction
   misattribution in prose**, invisible to the direction-integrity guard (which checks
   lean-vs-prose only, and correctly returned Consistent). the first candidate edge
   signal data point is therefore polluted: it measures a misread, not an edge attempt.
2. **the other five runs are market echo.** every lean equals the staged consensus side,
   and every summary cites market consensus as support. on close favorites priced
   0.51-0.57, echoing the market converges to the market's own hit rate; the market went
   1-4 on those five games that night. expected wins on the five agree runs at staged
   probabilities ≈ 2.7; DAI got 1. P(<=1 win) ≈ 0.15 -- poor but inside small-sample
   variance for a coin-flip-heavy slate.
3. **confidence is a template, not a measurement.** all six: exactly two grounded signals
   (starting_pitching, market), both quality=usable / support_cautiously,
   evidenceRichness=2, sourceSufficiency band=moderate, posture=monitor, advertised High,
   analyzerConfidence == confidence (no dampening). 0.75 was assigned five times and 0.80
   once (when "pitching advantage AND market support" aligned -- the 0.80 run lost).
   confidence tracked narrative coherence, not evidence strength; staged win probabilities
   were 50-57%.

direct answers to the slice questions: (Q1) primarily normal small-sample variance ON TOP
OF two structural issues -- market echo and one confirmed market-interpretation error; not
bad data (staged data was correct). (Q2) leans skewed home 5/6, but 4 of those 5 echoed a
home-favoring slate; the bias is market-echo-shaped, not an independent home preference in
this cohort (globally home leans do underperform: 0.557 n=70 vs 0.643 n=28). (Q3) lean/
prose direction was Consistent on all six; the mismatch that DOES exist is prose-vs-staged
-market-direction on 823036 -- a class the current guard does not check. (Q4) the model
overvalued season-aggregate starter stats and market consensus (the only two signals it
had); no line movement, injuries, lineup form, or rest data existed to weigh. (Q5) no --
confidence did not track evidence quality; it is structurally uniform at 0.75/0.80 for
this regime. (Q6) see section 7; the only immediately actionable item is read-only.

## 3. cohort reconstruction table

| gamePk | matchup | dai lean (conf) | dai's stated basis | staged market (consensus, homeP) | market agree | final | result | eval | apparent reason for lean |
|---|---|---|---|---|---|---|---|---|---|
| 822958 | NYY@TB | home TB (0.75) | starter edge (Schlittler>Jax) + market | home, 53% | yes | NYY 5-1 | away | INCORRECT | starter season stats + consensus echo |
| 822712 | HOU@WSH | home WSH (0.75) | market consensus + home field; starters similar | home, 54% | yes | WSH 12-11 | home | **CORRECT** | consensus echo (explicitly market-led) |
| 824900 | NYM@ATL | home ATL (0.80) | starter edge (López>Peralta) + market | home, 57% | yes | NYM 7-6 | away | INCORRECT | both signals aligned -> 0.80 |
| 823036 | MIL@STL | home STL (0.75) | starter edge (Drohan>May) + "market prefers Cardinals" | **away, 50%** | **NO** | MIL 4-3 | away | INCORRECT | starter edge + MISREAD market as agreeing |
| 823282 | ARI@SD | home SD (0.75) | market consensus + Buehler "experience"; both ERAs elevated | home, 53% | yes | ARI 8-0 | away | INCORRECT | consensus echo on weak starter case |
| 823205 | TOR@SF | away TOR (0.75) | starter edge (Gausman>Roupp) + market | away, 51% | yes | SF 10-1 | home | INCORRECT | starter stats + consensus echo |

uniform on all six: groundedSignals = [starting_pitching, market] exactly;
missingSignals = [] (understated -- see H5); bullpen_availability shallow proxy;
evidenceRichness 2; posture monitor; advertisedStrength High; directionConsistency
Consistent; settlement provenance complete.

## 4. pattern analysis

- **side bias**: 5/6 leans home, but 4/5 agree-leans echoed a home-favoring consensus;
  the away lean (823205) also echoed consensus. within-cohort side bias is market-echo
  shaped. context: pooled home-lean accuracy 0.5571 (n=70) vs away 0.6429 (n=28) --
  a real global pattern worth tracking, not attributable from n=6.
- **market relationship**: agree 5 (1 correct), accidental-disagree 1 (incorrect). DAI
  functioned as a market echo with a starter-stats tiebreaker. favorites lost broadly on
  this slate (three games flipped after the blocked-settlement snapshot).
- **confidence**: 0.75 x5 (1 correct), 0.80 x1 (incorrect). the 0.80 was awarded for
  signal AGREEMENT ("pitching advantage and market support"), i.e., coherence -- exactly
  the mechanism the projection flagged behind the global gte_0.80 inversion (0.50 n=16).
- **evidence profile**: identical on all six -- the correct run is indistinguishable from
  the five misses on every structured field. nothing in the artifact separates the win
  from the losses; that is itself the finding (no discriminating evidence signal exists
  in this regime's current vocabulary).
- **feature weighting** (from prose): season-aggregate starter ERA/WHIP dominated; market
  consensus used as confirmation; home field mentioned twice as a bonus; bullpen (shallow
  proxy), injuries, lineup form, rest/travel, line movement never available and never
  weighed. no over-weighting of an exotic signal -- rather uniform over-reliance on the
  only two signals staged.
- **prose-to-lean consistency**: all six Consistent (guard verified). counter-cases are
  present but formulaic one-liners; none undermined its lean. the 823036 counter-case
  ("May's home performance") also misplaces the AWAY starter at home -- a second
  referential error in the same artifact.
- **market-attribution consistency (new class)**: 5/6 prose market claims match staged
  consensus; 823036 does not. no current check compares the artifact's market claim to
  the staged/persisted consensus side.

## 5. per-run failure classification

| gamePk | classification |
|---|---|
| 822958 | market-aligned miss; reasonable at available evidence; variance |
| 822712 | correct by 1 run; consensus echo -- no skill evidence either way |
| 824900 | reasonable miss + **confidence inflation instance** (0.80 for coherence, lost by 1) |
| 823036 | **evidence-interpretation error**: market-direction misattribution in summary + discern + counter-case starter misplacement; accidental divergence; 1-run loss |
| 823282 | market-aligned miss; weakest stated case ("experience") already contradicted by its own counter-case; blowout 0-8 |
| 823205 | market-aligned miss; blowout 1-10; nothing in artifact warned of it |

## 6. root-cause hypotheses (ranked)

**H1 -- market-direction misattribution can occur in artifact prose (CONFIRMED, 1 of 6).**
evidence: 823036 staged `consensus away` vs summary/discern "market prefers Cardinals."
confidence in hypothesis: high (direct artifact-vs-staged-data comparison). confirm: a
read-only corpus scan comparing every artifact's market claim to persisted
marketConsensusSide. falsify (as systemic): scan finds no other instances. testable:
READ-ONLY NOW. status: allowed (diagnostic); a deterministic market-attribution guard
(analogous to the direction-integrity guard) would be a later controlled code slice.

**H2 -- backed_depth reads are market echo; no independent edge mechanism exists yet.**
evidence: 12 of 13 settled registry backed_depth leans equal consensus; all six summaries
cite consensus as support; the lone structural disagreement was H1's accident.
confidence: medium-high. confirm: disagreement sample growth shows leans that KNOWINGLY
oppose consensus with distinct reasoning. falsify: future artifacts show explicit
"market disagrees with this read because..." reasoning. testable: with future captures
(already the cadence program). status: allowed to measure; any prompt change to force
market-relationship awareness is a later controlled slice.

**H3 -- confidence is template-driven (coherence score), inflated relative to evidence.**
evidence: uniform 0.75 default with two usable signals; 0.80 awarded for signal
agreement; aConf==conf (no dampening); staged win probs 50-57%; global gte_0.80 band
0.50 (n=16), inverted under the live criterion. confidence: high that the MECHANISM is
template-like; medium that it is the cause of the inversion (n=16). confirm: inversion
persists as gte_0.80 n grows; falsify: 0.80-band accuracy recovers above the bar.
testable: with future settled evidence. status: confidence-rule changes LOCKED behind
gate-4/criterion review; doctrine already labels confidence provisional/uncalibrated.

**H4 -- small-sample variance did the raw damage.** evidence: expected ~2.7 wins on the
five agree runs at staged probabilities; P(<=1) ≈ 0.15; two losses were 1-run games; the
market itself went 1-4 on the agree games. confidence: high that variance explains most
of the 1/6 magnitude GIVEN the echo structure. confirm/falsify: only more settled n.
status: the correct response is exactly what the cadence does -- keep capturing and
settling, change nothing from n=6.

**H5 -- signal vocabulary in this regime is thin and understates its own gaps.**
evidence: only 2 grounded signals; bullpen shallow proxy; missingSignals=[] on every
artifact while discern prose itself notes "lineup form data is missing"; evR frozen at 2.
confidence: high (structural, directly observable). confirm: n/a -- it is a fact; the
question is impact. owner: perceive (retrieval scope) + discern (availability model).
status: data-source improvements are candidate-later controlled slices.

**H6 -- season-aggregate starter comparison is too shallow a tiebreaker.** evidence: all
starter cases argued from season ERA/WHIP only (that is all the staged detail contains);
823282's stated case was "experience." confidence: medium. testable: only with enriched
retrieval (candidate later). status: locked behind data-source work; not tunable now.

## 7. improvement backlog (no implementation in this slice)

**allowed now (read-only / process):**
- corpus-wide market-attribution audit: compare every settled artifact's prose market
  claim to persisted marketConsensusSide; tag mismatches with exclusion-style rationale
  in a diagnostic report (H1 scope; read-only scan).
- failure taxonomy in settlement readouts: add the section-5 classification table as a
  standard per-cohort readout appendix (report format only).
- mandatory divergence post-mortem: every marketAgreement=false row gets an H1-style
  staged-vs-prose check at settlement time, in the readout (process rule, no code).
- claim-safe posture unchanged: buyer copy stays free of accuracy/edge claims.

**candidate later (controlled experiment / approved slice):**
- market-attribution consistency guard (deterministic, settlement- or compose-time),
  modeled on the direction-integrity guard -- flags or refuses when artifact prose
  asserts a market direction contradicting the persisted consensus.
- prompt hardening: require the artifact to state the market side explicitly and its
  relationship to the lean (agree/disagree + why) -- prompt change, gated.
- confidence dampening review (docs-only criterion review first; implementation locked
  behind gate 4).
- market-agreement-aware stance dampening; posture thresholds review.
- retrieval enrichment: lineup form, injuries, rest/travel, line movement (fallback-ladder
  discipline applies: prefer same-signal recovery over lateral proxies).
- model comparison under an evidence plan.

**forbidden now:**
- tuning anything from 6 rows.
- changing gate criteria to pass (gate 4 stays FALSE on merits).
- buyer-facing accuracy/edge/win-rate claims.
- registry default-on.
- threshold edits without controlled review.
- model replacement without an evidence plan.

## 8. what this does not license

- no tuning
- no threshold edits
- no model replacement
- no prompt changes
- no buyer-facing accuracy, edge, or performance claims
- no gate edits
- no registry default-on change
- no new capture authorization

divergence rows remain **candidate edge signal** only -- and this analysis shows the
first one was not even a genuine edge attempt, which strengthens, not weakens, the case
for the claim-safe posture.

## 9. recommended next slice

**Diagnostic Readout / Failure Taxonomy v1** (read-only): the clear measurement gap is
H1 -- run the corpus-wide market-attribution audit (every settled artifact's market claim
vs persisted consensus side), and standardize the per-run failure classification as a
readout appendix. this also decides whether the 823036 row deserves a diagnostic
annotation in future readouts (it stays in the binding denominator -- it settled
legitimately -- but the candidate-edge ledger should note it was an accidental
divergence). NOTE: if the 2026-07-07 capture cohort becomes Final first, settle that
cohort before any further analysis or capture (its own blocked-attempt handoff governs);
its divergence run 824820 should get the H1 staged-vs-prose check at settlement.
