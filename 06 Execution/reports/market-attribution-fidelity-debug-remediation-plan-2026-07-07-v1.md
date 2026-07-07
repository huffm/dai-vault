---
title: "Market Attribution Fidelity Debug + Remediation Plan v1 (2026-07-07)"
type: "report"
date: "2026-07-07"
status: "COMPLETE -- root cause located (staged-correct / prompt-hazardous / prose-wrong); guard-first remediation recommended"
project: "DAI"
slice: "Market Attribution Fidelity Debug + Remediation Plan v1"
related:
  - "06 Execution/reports/diagnostic-readout-failure-taxonomy-2026-07-07-v1.md"
  - "06 Execution/reports/backed-depth-prediction-failure-analysis-2026-07-07-v1.md"
---

# market attribution fidelity debug + remediation plan v1

## 1. purpose

locate exactly where staged market truth becomes incorrect artifact prose, then choose
the minimal safe remediation. debug/docs only -- no code, prompt, gate, or capture change
occurs in this slice.

## 2. executive summary

**the staged data is correct at every persisted step; the defect happens inside the
model's generation, enabled by two specific prompt-context design hazards.**

1. the market overlay never names the consensus TEAM. the model receives
   `moneyline consensus favors the away side.` (literal side label,
   `mlb.overlay.market.backed_depth.v1.txt` line 5 / `sports_analyzer.py` ~896) and must
   resolve "the away side" to a team while simultaneously forming its lean -- and on
   every persisted divergence it resolved it to its lean's team instead.
2. the numeric context is home-only, un-de-vigged, and rounded: the prompt shows
   `median implied win probability: {home team} {format_pct_whole(medianHomeImpliedProb)}%`
   -- the RAW per-side median (medians are not de-vigged: 823036 persisted home 0.500 AND
   away 0.537, sum > 1), home side only, rounded whole. both known-bad fixtures rendered
   as "**50%**" beside "consensus favors the **away** side" -- a numerically
   self-contradictory-looking block on exactly the close games divergence capture screens
   for.

corpus correlation (37 rows with medians): prompt blocks that LOOK incoherent
(displayed home pct >= 50 while consensus=away) misattributed **5/7 (71%)**; coherent
blocks misattributed 3/30 (10%). the incoherent display AGGRAVATES; the side-label
indirection is the base defect (three coherent-block rows still flipped: 824500 @47%,
824662 @40%, 824012 @43%).

remediation: **Option E staged hybrid -- deterministic Market Attribution Fidelity Guard
v1 first (no generation behavior change, quantifies the defect, protects the ledger),
then Prompt Market Context Hardening v1 as a separate controlled slice** (name the team
beside the side label; require an explicit market-vs-lean acknowledgment; fix the
numeric display) once the guard provides a before/after measurement baseline.

one taxonomy correction: 822956's "-1.5" language was NOT invented -- the prompt's run
line block (template line 2) carries real spreads from the odds source; that note in
diagnostic-readout-failure-taxonomy-2026-07-07-v1.md section 11 is withdrawn. market-TYPE
fidelity is not a defect class in evidence.

## 3. known fixtures

- 823036 MIL@STL: staged consensus away (Brewers), persisted medians home 0.500 / away
  0.537; prose + discern claim market prefers the Cardinals. accidental divergence.
- 824820 CHC@BAL: staged consensus away (Cubs), persisted medians home 0.505 / away
  0.535 (displayed "50%"); prose + discern claim market favors the Orioles. accidental
  divergence.
- clean controls: 823687 CLE@MIN (consensus home, displayed 54% -- prose correct),
  822712 HOU@WSH (consensus home, 54% -- prose correct).

## 4. source-to-prose trace (both fixtures identical in shape)

| step | field/source | expected mkt side | observed | status | notes |
|---|---|---|---|---|---|
| 1 retrieval | OddsMarketClient spread+depth readings | away | away | correct | 9 books, h2h + run line read |
| 2 staged facts | BaseballMarketContext / sourceDepth market_odds detail | away | away ("consensus away") | correct | detail string also shows the raw home median ("50%") |
| 3 derivation | Depth.ConsensusSide -> persisted marketConsensusSide (/rows) | away | away | correct | medians persisted per side, raw (not de-vigged) |
| 4 prompt serialization | market overlay lines (analyzer ~886-913 / template v1) | away | "favors the away side" + "{home team} 50%" | **technically correct, semantically hazardous** | side label never mapped to a team name; home-only raw rounded pct reads as tossup |
| 5 model output | summary/discern/counterCase JSON | away | **home team named as market-favored** | **INCORRECT -- mutation point** | model resolves "away side" to its lean's team |
| 6 artifact summary | persisted OutputJson.summary | away | home claim | faithful to step 5 | no c# rewriting of prose (protocol is model-produced; composer adds deterministic synthesize strings only, per the 07-05 audit) |
| 7 discern phase | cognitiveProtocol.discern.contrast | away | home claim | faithful to step 5 | |
| 8 counter-case | counterCase | away | consistent with wrong claim | faithful | 823036 also misplaces the away starter "at home" |
| 9 wwctr | whatWouldChangeTheRead | away | presumes market favors lean | faithful | |
| 10 read model | /rows marketConsensusSide/marketAgreement | away / false | away / false | correct | the persisted fields never lied; only prose does |

## 5. mutation-point analysis

- **1. source retrieval/staging** -- against: staged detail + persisted fields correct on
  all 38 audited rows. confidence it is NOT here: high. falsify: any row where /rows
  consensus disagrees with the raw book readings (none found).
- **2. market consensus derivation** -- against: consensus label matches the persisted
  per-side medians on every corpus row (higher raw median side = consensus side, incl.
  823036: away 0.537 > home 0.500). NOT here: high.
- **3. prompt context serialization** -- PARTIAL contributor, not mutator: the values
  injected are correct, but the representation is hazardous (side label without team;
  home-only raw rounded pct; "50%" rendering for 0.495-0.505). evidence: 71% vs 10%
  misattribution split by block coherence. confirm further: reproduce with the template
  offline. this is a design weakness, not a data bug -- fixing it is a PROMPT change.
- **4. model interpretation** -- **THE MUTATION POINT**: the model writes a market
  sentence naming the wrong team despite a correct label in context. evidence: all 9
  mismatches are wrong-TEAM claims; 3 occurred even with numerically coherent blocks;
  the flip always lands on the lean's team on divergence rows (7/7 staged-opposed rows
  incl. the no-lean canary's narrative side). confidence: high. falsify: finding a
  persisted prompt whose market block already named the wrong team (none exists -- the
  template cannot name a team).
- **5. artifact parsing / 6. composition / 7. read-model / 8. name resolution** --
  against: prose is persisted verbatim; /rows fields correct; team refs correct on all
  rows. NOT here: high.

## 6. root-cause conclusion

staged-correct / prose-wrong. the defect is a **model attribution failure at generation
time, materially enabled by prompt-context design**: a bare home/away side label the
model must map to a team mid-generation, plus a home-only raw rounded percentage that
renders as "50%" on exactly the close-favorite slates the divergence program targets.
no source mapping fix applies (Option D eliminated by evidence).

## 7. remediation options

| option | fixes | does not fix | blast radius | test burden | changes generation? | paid-free? | verdict |
|---|---|---|---|---|---|---|---|
| A deterministic post-artifact guard | detection, classification (accidental/deliberate/unclear), ledger credit protection, readout automation | the prose itself; buyer-visible wrong claims keep being generated | small (read-side evaluator + additive /rows field) | moderate (fixtures below) | no | yes | **recommended first** |
| B prompt/context hardening | future prose errors at the source (team-named consensus, required market-vs-lean acknowledgment, de-vigged both-side display) | already-generated artifacts; needs paid canary validation | medium (prompt bytes change -> registry byte-equality invariant + recipe reversioning; behavior shift on every future run) | high (canary + cohort A/B vs guard baseline) | YES | validation needs paid canaries | recommended SECOND, separate approval-gated slice |
| C compose-time correct/fail | prevents invalid artifacts shipping | root cause; adds fail/hold/regeneration complexity; a "correction" that rewrites model prose deterministically is a new integrity risk | large | high | effectively yes (artifact lifecycle) | partially | not recommended now; revisit only if B fails to cut the rate |
| D source mapping fix | nothing -- sources are correct | n/a | n/a | n/a | n/a | n/a | eliminated by evidence |
| E staged hybrid (A then B) | detection + ledger integrity now; prevention next, measured against the guard's baseline | nothing material | small then medium | staged | no, then controlled yes | guard yes | **RECOMMENDED** |

## 8. recommended next implementation

**Market Attribution Fidelity Guard v1** (Option A, the first stage of E):

1. deterministic evaluator (c#, beside SettlementDirectionIntegrity): inputs = persisted
   marketConsensusSide + home/awayTeamRef + artifact prose fields (summary, counterCase,
   lean, whatWouldChangeTheRead, cognitiveProtocol discern strings from OutputJson).
   resolve team mentions in market-referencing sentences via team-ref nicknames (the
   audit's method, verified on 38/38 manually); compare claimed side vs persisted side.
2. outputs: PASS | FAIL_MARKET_ATTRIBUTION_MISMATCH | UNCLEAR_MARKET_ATTRIBUTION, plus
   divergence class on marketAgreement=false rows: deliberate (prose correctly
   acknowledges opposition) / accidental (mismatch) / unclear.
3. surface additively on /rows (e.g. marketAttributionStatus, divergenceClass); never
   blocks settlement; never calls a model; /metrics byte-identical; gate-4 persisted
   counts untouched -- the ledger and readouts consume the classification.
4. known nickname hazards to handle: multi-word nicknames (white-sox, red-sox, blue-jays),
   pronoun-only market sentences -> UNCLEAR, both-teams-named sentences -> resolve by
   claim verbs or return UNCLEAR (824175's self-contradiction must land UNCLEAR).

then, as a SEPARATE approval-gated slice, **Prompt Market Context Hardening v1**:
consensus line names the team ("favors the away side (Milwaukee Brewers)"), adds a
required market-vs-lean acknowledgment to the output contract, and displays de-vigged
both-side percentages -- validated by paid canary against the guard's baseline rate.
prompt bytes change means registry recipe reversioning; that slice owns the invariants.

## 9. test plan (for the guard slice)

fixtures (real persisted artifacts + synthetics):
- 823036 -> FAIL_MARKET_ATTRIBUTION_MISMATCH, divergenceClass=accidental.
- 824820 -> FAIL_MARKET_ATTRIBUTION_MISMATCH, divergenceClass=accidental.
- clean agree control (e.g. 823687 or 822712 artifact) -> PASS.
- synthetic deliberate divergence (staged away, lean home, prose: "the market favors the
  Brewers, but this read leans Cardinals because ...") -> PASS, divergenceClass=deliberate.
- synthetic ambiguous (prose asserts both sides, modeled on 824175) -> UNCLEAR.
- agree-row inversion (824012 artifact: prose names the wrong team ON an agree row) ->
  FAIL, no divergence class (marketAgreement=true).

invariant tests: gate-4 pooled counts identical before/after (pooled_calibration on the
same /rows dump); settlement path untouched (guard result never consulted by /reconcile);
zero model calls (pure function over persisted fields); determinism (same input -> same
output, twice); accidental/unclear rows excluded from any candidate-edge ledger view;
/metrics byte-identical.

## 10. paste-ready implementation prompt

```text
SLICE: Market Attribution Fidelity Guard v1 (implementation)

Mode: code-allowed (dai), TDD required, no paid model calls, no captures, no
reconciliation, no settlement writes, no prompt/routing/confidence/gate criteria
changes, registry stays default-off, push after verification, no force push.

Objective: implement the deterministic market-attribution fidelity guard specified in
market-attribution-fidelity-debug-remediation-plan-2026-07-07-v1.md section 8 and the
taxonomy report section 8. detection + classification only; it never blocks settlement,
never calls a model, and never changes gate-4 persisted counts.

Read first: the debug report (sections 4-9), diagnostic-readout-failure-taxonomy-2026-
07-07-v1.md, SettlementDirectionIntegrity + ArtifactDirectionConsistencyInput (the
existing analogous guard), PromptRouteCalibrationExport.cs (/rows additive-field
precedent), the six persisted divergence artifacts.

Implement:
1. MarketAttributionFidelity evaluator (DevCore.Api, beside SettlementDirectionIntegrity):
   pure, deterministic; inputs = marketConsensusSide, homeTeamRef, awayTeamRef, and the
   artifact prose fields (summary, counterCase, lean, whatWouldChangeTheRead, discern
   strings) parsed fail-soft from OutputJson. team mention resolution by ref nicknames
   incl. multi-word nicknames; pronoun-only -> UNCLEAR; both-sides-asserted -> UNCLEAR.
2. outputs: PASS | FAIL_MARKET_ATTRIBUTION_MISMATCH | UNCLEAR_MARKET_ATTRIBUTION; on
   marketAgreement=false rows also divergenceClass deliberate|accidental|unclear
   (deliberate = prose correctly names the opposing consensus team while explaining the
   lean).
3. surface additively on /rows as marketAttributionStatus + divergenceClass. no /metrics
   change (assert byte-identical). no schema migration if derivable at read time.
4. TDD with the fixtures in the debug report section 9 (823036/824820 FAIL+accidental,
   clean control PASS, synthetic deliberate PASS+deliberate, 824175-style UNCLEAR,
   824012 agree-row FAIL) + invariant tests (no settlement coupling, determinism, gate
   counts unchanged, no model call).
5. full DevCore.Api.Tests suite green; agent-service suite untouched/green; live check:
   GET /rows shows the new fields with 6/6 persisted divergences classified accidental.

Then: dai-code-reviewer review, docs-only vault note updating the taxonomy ledger
language to consume the new fields, commits (dai: feat(sports): add market attribution
fidelity guard; dai-vault: docs(calibration): wire attribution guard into ledger
discipline), push after verification, 13-section handoff. capture cadence stays PAUSED
until this guard is live and the readout language consumes it. do not touch prompts --
Prompt Market Context Hardening v1 is a separate approval-gated slice.
```

## 11. what this does not authorize

- no guard implementation in this slice (plan only)
- no prompt changes, no tuning, no threshold edits, no model replacement
- no gate edits, no registry default-on change
- no new captures, no cadence resume (still PAUSED)
- no settlement of the 2026-07-07 cohort (its readiness guard governs)
- no buyer-facing accuracy, edge, win-rate, or performance claims
- accidental divergences remain non-candidate-edge

## 12. recommended next slice

root cause is staged-correct/prose-wrong -> **Market Attribution Fidelity Guard v1**
(section 10 prompt) is next, UNLESS the 2026-07-07 cohort reaches READY first (run
check-settlement-finals.ps1; on READY, the taxonomy-aware settlement runs first,
preserving the accidental label for 824820). after the guard is live and measured:
Prompt Market Context Hardening v1 (separate, approval-gated, paid-canary-validated).
