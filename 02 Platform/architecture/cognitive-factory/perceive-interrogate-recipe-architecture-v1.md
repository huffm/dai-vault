# perceive / interrogate / prompt recipe architecture v1

**status:** draft doctrine -- proposed architecture; each implementation slice remains individually approved. the deterministic Market Attribution Fidelity Guard (dai `a0db824`) is the shipped existence proof of the pattern; everything else here is design, not built machinery.
**date:** 2026-07-07

## 1. executive summary

DAI generates decision artifacts from staged facts, and the platform just proved -- with a
settled failure corpus -- that staged facts alone are not enough. the artifact prose can
contradict the very data it was given.

the pattern this document architects has three parts:

- **Perceive establishes source truth.** before any generation or audit, the platform
  binds the run's decisive facts into an explicit source-truth contract: not "the data is
  in context somewhere" but "these named facts are binding, and here is each one."
- **Interrogate tests whether reasoning and prose faithfully follow that truth.** every
  interrogation is performed by an analytical role with a contract: what it checks, what
  facts bind it, what contradictions it hunts, what classifications it must emit, and
  what it is forbidden to infer or soften.
- **prompt recipes provide the analytical role used for each interrogation.** the prompt
  builder composes each interrogation (and, eventually, generation) from reusable pieces
  -- role recipe, source-truth contract, evidence payload, interrogation pattern, output
  contract, failure semantics -- instead of hand-writing a monolithic prompt per task.

the one-line version: **data alone does not constrain a model; the system must also hand
the model an interpretive frame that says how to examine the data, and then check that
the output obeyed it.**

## 2. problem statement

the market attribution defect (debug report 2026-07-07, taxonomy 2026-07-07) is the
motivating failure, and it is fully diagnosed:

- retrieval, staging, consensus derivation, persistence, and the read model were all
  CORRECT on every audited row. the persisted `marketConsensusSide` and
  `marketAgreement` never lied.
- the model received technically correct but semantically weak market context: a bare
  side label -- `moneyline consensus favors the away side.` -- never a team-bound
  statement like "the market favors the away Cubs." it also received a home-only,
  un-de-vigged, whole-rounded median that rendered as "50%" on exactly the close games
  the divergence program targets.
- mid-generation, while building a narrative for its lean, the model resolved "the away
  side" to the wrong team on every staged-opposed row audited: all six persisted
  marketAgreement=false rows in the taxonomy corpus were ACCIDENTAL divergences (prose
  claimed market support for DAI's lean); zero deliberate divergences existed among
  settled valid rows. ~24% of the 38-artifact corpus misattributed the market somewhere
  in prose, including on agree rows.
- consequence one: the candidate-edge ledger was measuring an attribution defect, not
  contrarian judgment. consequence two: buyer-visible prose contained false market
  claims. consequence three: nothing in the pipeline could see it -- the existing
  direction-integrity guard compares lean to prose, never prose to staged market.

the deterministic Market Attribution Fidelity Guard v1 now detects this class after
generation and protects interpretation (ledger credit, readout language). it does not
prevent the generation failure, and it is one hand-built check for one failure class.
this architecture generalizes what the guard proved: source truth must be bound
explicitly, and fidelity to it must be interrogated by role-specific contracts.

## 3. perceive stage: the source-truth contract

Perceive is the stage that converts retrieved data into a bound, named, per-run fact set
-- the **source-truth contract**. it binds; it never interprets.

for sports market attribution, the contract carries facts like:

- matchup (away @ home), home team, away team
- DAI lean side AND DAI lean team (both -- side labels without team bindings are exactly
  what failed)
- market consensus side AND market consensus team
- marketAgreement (true/false) -- stated, not left for anyone to re-derive
- median home probability; median away probability when available; whether the medians
  are de-vigged or raw (the current raw per-side medians sum above 1.0 -- a bound fact
  must say which it is)
- book count; market type (h2h moneyline vs run line vs spread)
- the explicit sentence of relationship: "DAI disagrees with the market: DAI leans home
  (Cardinals); the market favors away (Brewers)."

rules:

- every fact is team-bound where a side exists. "away" is a coordinate; "away
  (milwaukee-brewers)" is a fact.
- the contract is derived deterministically from persisted/staged fields -- the same
  fields /rows already carries. no model call produces it.
- the contract is the SINGLE source both generation and interrogation cite. the
  generation prompt's evidence payload is rendered FROM it; the interrogation recipes
  check prose AGAINST it. one derivation, two consumers, no drift.
- Perceive does not weigh, soften, rank, or conclude. interpretation belongs to discern
  and decide; fidelity checking belongs to interrogate.

alignment with existing doctrine: this is a sharpening of decision 0004's perceive
(detect / frame / aim), not a new stage. today the "contract" exists implicitly as
persisted columns (LeanSide, MarketConsensusSide, medians, team refs) plus prompt
serialization that re-renders them weakly. the change is to make the binding explicit
and team-bound, and to feed generation and interrogation from the same rendering.

## 4. interrogate stage: role-driven fidelity checks

Interrogate applies role-specific analytical checks against the source-truth contract.
decision 0004 gives interrogate three micro-actions (question, probe, verify); this
architecture concretizes **verify** as a set of recipe-driven checks.

for the market attribution class, the interrogation asks:

- does the generated prose name the same market side (team) as the staged market data?
- if DAI disagrees with the market, does the prose explicitly acknowledge that the
  market favors the other side?
- does the prose accidentally claim the market supports DAI's lean?
- does any section assert both market directions?
- classification: deliberate divergence, accidental divergence, unclear divergence, or
  market aligned?

two execution modes, one contract shape:

- **deterministic recipes** (preferred wherever the check is mechanically decidable):
  pure code over persisted fields. the shipped fidelity guard is exactly this -- the
  Market Attribution Auditor recipe hard-coded as `MarketAttributionFidelity.cs`.
  deterministic recipes cost nothing, run on every row, and never hallucinate.
- **model-run recipes** (only where judgment is genuinely required): a composed prompt
  giving a model the role contract + source-truth contract + evidence and demanding the
  classification schema. model-run recipes are paid, approval-gated, and their outputs
  are diagnostics -- never evidence, never gate inputs.

the preference order is absolute: if a check can be deterministic, it must be. a
model-run interrogation of model output is a last resort, not a default.

## 5. prompt recipe architecture

a **role recipe is an analytical contract, not a persona.** "you are a skeptical
analyst" constrains nothing; a recipe constrains everything that matters:

- **recipe name** -- stable identifier, versioned (registry-style, like the existing
  prompt recipe ids: `interrogate.market_attribution_auditor.v1`).
- **analytical role** -- what kind of analysis this is (fidelity audit, confidence risk
  classification, contradiction sweep).
- **purpose** -- the one question the recipe answers.
- **required source facts** -- the named source-truth-contract fields that BIND the
  analysis. missing required facts -> UNCLEAR, never a guess.
- **interrogation questions** -- the ordered checks, phrased as decidable questions.
- **allowed inferences** -- what the role may conclude from the facts.
- **forbidden inferences** -- what it must never do: soften a mismatch into "nuance,"
  infer market direction from probabilities the contract marks ambiguous, treat a
  hypothetical ("what would change the read") as a current claim, upgrade unclear to
  pass.
- **output schema** -- the exact classification enum + reason + evidence fields.
- **pass/fail/unclear semantics** -- what each status licenses downstream, and the rule
  that unclear is the honest answer when the inputs cannot be compared safely.
- **examples** -- at least one worked pass, fail, and unclear.
- **test fixtures** -- REAL failure fixtures from the corpus (823036, 824820) plus
  synthetics; a recipe without fixtures does not ship. the fidelity guard's 17 tests
  are the template.

## 6. example recipe: market attribution auditor

- **recipe name:** interrogate.market_attribution_auditor.v1
- **analytical role:** source-vs-prose fidelity audit (market direction).
- **purpose:** does the artifact prose faithfully represent the staged market direction,
  and how should the persisted divergence be interpreted?
- **required source facts:** marketConsensusSide + consensus team, DAI leanSide + lean
  team, home/away team bindings, marketAgreement; prose sections lean / summary /
  discern contrast / discern weigh (counter-case and whatWouldChangeTheRead are
  inspected but are never claim sources -- one is counter-by-nature, the other
  hypothetical-by-nature).
- **interrogation questions:** section 4's list, in order.
- **allowed inferences:** mapping a named team to its side via the contract's bindings;
  resolving one market claim per clause.
- **forbidden inferences:** guessing a side from probabilities alone; reading a
  hypothetical shift as a claim; softening a flipped market into "the market is close";
  counting an accidental divergence as edge under any narrative.
- **output schema:** attributionFidelityStatus (PASS | FAIL_MARKET_ATTRIBUTION_MISMATCH
  | UNCLEAR_MARKET_ATTRIBUTION); divergenceInterpretation (market_aligned |
  deliberate_divergence | accidental_divergence | unclear_divergence); reason (bounded
  snake_case); evidence quotes (prose clause + staged fact); countsAsCandidateEdge
  (true only for deliberate_divergence).
- **semantics:** deliberate divergence may count as candidate edge; accidental gets zero
  edge credit; unclear gets zero credit until reviewed; market aligned is not a
  divergence; none of this changes persisted gate counts.
- **status:** SHIPPED in deterministic form (dai `a0db824`; /rows fields
  attributionFidelityStatus / attributionFidelityReason / divergenceInterpretation;
  pattern doc `06 Execution/patterns/market-attribution-fidelity-guard-v1.md`). live
  baseline 2026-07-07: Pass 72 / FAIL 10 / Unclear 203 over 285 rows; 6/6 known
  accidentals confirmed; 4 true deliberate divergences discovered in legacy rows.

## 7. example recipe: confidence skeptic

- **recipe name:** interrogate.confidence_skeptic.v1
- **analytical role:** confidence-risk classification (never tuning).
- **purpose:** is the artifact's stated confidence justified by the source-truth
  contract's evidence quality, or inflated by coherent-sounding but weak signals?
- **required source facts:** confidence, analyzerConfidence, evidenceRichness, grounded
  vs missing signals, source depth per group (enriched vs shallow proxy), market
  relationship (agree/disagree + gap), posture.
- **interrogation questions:** is confidence supported by evidence richness, or is the
  richness score frozen while confidence varies? is confidence higher when signals
  merely AGREE with each other (coherence) rather than when they are independently
  strong? did the artifact choose a stronger stance than the source facts justify --
  should this read have been monitor or wait rather than a directional lean at 0.75+?
  does the stated confidence exceed what the staged market probability (a bound fact)
  would imply for a coin-flip game?
- **allowed inferences:** classifying confidence risk (justified / coherence_inflated /
  unsupported / unclear) with named evidence.
- **forbidden inferences:** proposing new thresholds, recomputing confidence, editing
  dampening, recommending "just lower it" -- confidence rules are locked behind gate 4.
- **output schema:** confidenceRiskStatus + reason + the specific facts cited.
- **evidence basis for this recipe existing at all:** the failure analysis found
  confidence template-driven (uniform 0.75 with two usable signals; 0.80 awarded for
  signal coherence -- the 0.80 run lost; evidence profile identical on the win and the
  losses), and the live criterion shows the gte_0.80 region inverted. n remains small;
  this recipe CLASSIFIES risk, it does not conclude miscalibration.

## 8. example recipe: contradiction hunter

- **recipe name:** interrogate.contradiction_hunter.v1
- **analytical role:** internal-consistency sweep across artifact sections.
- **purpose:** find places where the artifact disagrees with itself or with its bound
  facts.
- **checks:** summary vs discern (do they claim the same market side, the same starter
  edge?); discern vs decide (does the resolved stance follow the weighed signals?);
  counter-case vs final stance (does the counter-case actually undermine the lean while
  the stance stays maximally confident?); source facts vs prose (every bound fact
  named in prose must match the contract -- the generalization of the market auditor to
  starters, scores, venues; the 823036 artifact also misplaced the away starter "at
  home"); lean vs market explanation (an agree-row prose that describes the market as
  opposing the lean, like 824012, is a contradiction even though nothing diverged).
- **output schema:** list of contradiction findings, each with section pair, quotes,
  bound fact, severity (blocking-for-interpretation vs note), plus an overall
  consistency status.
- **existing coverage to reconcile with, not duplicate:** the direction-integrity guard
  (lean vs prose direction) and the market attribution auditor are the first two cells
  of this matrix, both deterministic. contradiction hunter is the umbrella recipe;
  implement additional cells only when a settled failure demonstrates the class, per
  the no-overbuild rule.

## 9. prompt builder / orchestrator design

the orchestrator selects which interrogation recipes run for a given artifact, from
deterministic routing rules over the source-truth contract -- never model whim, never
tool selection by the model (tool-gateway doctrine holds):

- marketAgreement=false -> ALWAYS apply market attribution auditor (deterministic; runs
  today on every row regardless).
- confidence >= 0.80 -> apply confidence skeptic (the inverted region is exactly where
  scrutiny pays).
- source richness moderate-or-lower / any shallow proxy in a decisive signal -> apply a
  source fidelity auditor (does prose overstate shallow data?).
- artifact contains candidate-edge language -> apply buyer safety reviewer (claims
  discipline before anything buyer-visible).
- post-settlement incorrect -> apply the failure taxonomy recipe (classify the miss:
  market-aligned miss, accidental divergence, confidence inflation instance, variance).

routing rules are configuration, versioned like recipes. every applied recipe's output
lands on the diagnostic surface (/rows or the artifact inspection endpoint), additively,
exactly as the fidelity guard's fields do. no recipe output blocks settlement; no recipe
output enters gate-4 counts.

## 10. prompt composition pattern

the general shape of any composed interrogation (or hardened generation) prompt, in
order:

1. **system doctrine** -- the platform's standing rules (claim safety, no fabrication,
   posture vocabulary).
2. **role recipe header** -- the analytical contract: role, purpose, allowed and
   forbidden inferences.
3. **source-truth contract** -- the bound facts, team-bound, each named. binding means:
   if prose and contract disagree, the contract is right and the disagreement is the
   finding.
4. **evidence payload** -- the material under examination (artifact prose sections, or
   for generation: the staged evidence rendered FROM the contract).
5. **interrogation instructions** -- the ordered decidable questions.
6. **output schema** -- the exact classification structure; nothing free-form outside it.
7. **failure semantics** -- what FAIL and UNCLEAR mean downstream, and that UNCLEAR is
   the required answer when the comparison is unsafe.
8. **forbidden claims** -- what may never be asserted regardless of findings (edge,
   accuracy, buyer performance).

the same composition serves generation hardening: the sports market overlay's fix
(team-bound consensus line + required market-relationship acknowledgment) is precisely
"render the evidence payload from the source-truth contract and demand an output field
that acknowledges the bound relationship."

## 11. where this fits in the DAI pipeline

retrieve -> perceive -> interrogate -> discern -> decide -> synthesize -> evaluate ->
settle -> learn

- **retrieve** collects data (platform work; statsapi, odds, source-readiness; decision
  0004 keeps retrieval out of cognition).
- **perceive** binds facts into the source-truth contract (this doc, section 3).
- **interrogate** tests source/prose/reasoning integrity via role recipes (section 4);
  today: question+probe live in the analyze call, verify exists as the two deterministic
  guards; this architecture grows verify by recipe, mostly read-side.
  *(current-truth correction, 2026-07-21, WI-0036 Slice 1 -- verified in source: the line
  above misstates the split. `interrogate.question` and `interrogate.verify` are the
  model-emitted fields inside the shared analyze call; `interrogate.probe` is
  deterministic at compose time via `CognitiveProtocolBuilder.BuildProbe` over
  `SignalFollowUps` and is NOT part of the analyze call. The two deterministic guards
  (direction-integrity, market-attribution fidelity) are additional verify-class checks
  beside the model-emitted verify field, exactly the growth path this section proposes.
  The original text is preserved for its date. See decision
  [[0011-orchestrated-interrogate-perceive-refresh-loop-v1]] for the governed
  Question -> Probe -> authorized retrieval -> Perceive refresh -> Verify -> Discern
  target loop and [[wildcard-evidence-discovery-loop-v1]] for the proposal-only
  `SignalNeedProposal` contract that carries interrogation-discovered signal needs.)*
- **discern** weighs signals (unchanged; model-owned).
- **decide** chooses stance (unchanged; posture vocabulary intact).
- **synthesize** writes buyer-safe output (unchanged; claim discipline enforced).
- **evaluate** compares outcome after settlement (RunEvaluator; unchanged).
- **settle** writes outcomes idempotently with provenance (residue contract; unchanged;
  recipes never block it).
- **learn** turns failures into system-hardening requirements. currently this is manual
  slice work -- exactly the failure-analysis -> taxonomy -> debug -> guard chain of
  2026-07-07 -- and this architecture is that chain's generalization. "learn" is not an
  automated stage and must not become one without its own approved design.

## 12. implementation sequence

thin slices, each tied to a known failure, each individually approved. the 07-05 audit
verdict stands: do not build more protocol/interrogate MACHINERY speculatively; the
dormant cognitive-factory station runner stays dormant; probe-refresh stage 3 stays
locked behind gates 4/5. everything below is either read-side diagnostics (allowed
category) or an approval-gated generation change with a measured baseline.

1. **Prompt Market Context Hardening v1** -- the generation-side fix for the ONLY
   settled failure class: render the market evidence payload team-bound from the
   contract facts + require a market-relationship acknowledgment. approval-gated, paid
   canary, measured against the guard's live baseline (FAIL 10/285). this is the
   composition pattern's first generation-side instance, without any new machinery.
2. **Perceive Source Truth Contract v1** -- formalize the contract as a named, tested
   derivation (the fields already exist; this names and centralizes the rendering both
   generation and interrogation share). smallest useful form: one pure builder + tests;
   no schema change.
3. **Interrogate Recipe Registry v1** -- versioned recipe definitions + deterministic
   routing rules; starts with the one shipped recipe (market attribution auditor)
   registered retroactively. docs + thin config, not a runtime rewrite.
4. **Contradiction Hunter cells** -- add deterministic cells ONLY as settled failures
   demonstrate them (the starter-misplacement cell already has a fixture: 823036).
5. **Confidence Skeptic Recipe v1** -- read-side classification once the settled sample
   makes its outputs meaningful; never tuning.
6. **Post-Settlement Failure Taxonomy Recipe v1** -- standardize the per-run failure
   classification the 07-06 analysis did by hand, as a readout appendix generator.
7. **Prompt Builder Recipe Composition v1** -- generalize composition only after at
   least two recipes exist in registry form and a second niche or a second overlay needs
   it. building the composer before there are pieces to compose is the overbuild this
   sequence exists to prevent.

deferred decisions: model-run recipe execution (paid) has no approved use case yet;
"learn" automation; cross-niche recipe generalization; any buyer-facing surface for
recipe outputs.

## 13. what this does not authorize

- no model replacement
- no confidence tuning (locked behind gate 4)
- no gate edits (criterion stays as shipped; recipe outputs never enter gate counts)
- no buyer-facing performance claims (gate 5)
- no capture cadence resume until guard/readout discipline is in place (the fidelity
  guard pattern doc section 10 governs resume)
- no platform-wide abstraction that ignores the sports-specific failure evidence --
  every recipe ships against a real settled fixture from this corpus, or it does not
  ship
- no resurrection of the dormant cognitive-factory station machinery; no scheduler; no
  auto-settlement; registry routing stays default-off

truth hierarchy: source code, tests, and persisted fields outrank this document; this
document outranks per-slice improvisation about how interrogation should work. verify
against: `MarketAttributionFidelity.cs` + its tests (dai `a0db824`),
`ArtifactDirectionConsistency.cs`, `pooled_calibration.py` (gate authority),
`mlb.overlay.market.backed_depth.v1.txt` (the weak-context fixture), decision 0004
(cognitive protocol runtime), the 2026-07-07 debug/taxonomy/guard reports.

## 14. recommended next slice

**Prompt Market Context Hardening v1** -- not Perceive Source Truth Contract v1 first.
reasoning: the persisted fields already function as a de facto source-truth contract
(the shipped guard consumes them successfully), so formalizing the contract buys little
until something new consumes it; meanwhile the generation-side defect keeps producing
false market prose on every capture morning, buyer-visible, and now has a measured
baseline to beat (FAIL 10/285) and a fully specced fix (team-bound consensus line,
required market-relationship acknowledgment, de-vigged both-side display -- debug report
section 8). hardening IS the composition pattern's first real instance: it renders the
evidence payload from bound facts and demands an acknowledgment output. Perceive Source
Truth Contract v1 lands naturally as part of that slice's implementation (the shared
rendering), or immediately after as its formalization.

sequencing guard: the 2026-07-07 cohort settlement (readiness-guard-gated) and operator
approval still precede any generation change; prompt hardening changes prompt bytes and
therefore owns registry recipe reversioning + paid canary validation.

## related docs

- `02 Platform/architecture/cognitive-factory/cognitive-protocol-runtime.md` (decision 0004 shape)
- `02 Platform/architecture/sports-cognitive-worker-model.md` (signal vocabulary, pipeline)
- `06 Execution/patterns/market-attribution-fidelity-guard-v1.md` (shipped recipe instance)
- `06 Execution/reports/market-attribution-fidelity-debug-remediation-plan-2026-07-07-v1.md` (failure evidence + hardening spec)
- `06 Execution/reports/diagnostic-readout-failure-taxonomy-2026-07-07-v1.md` (corpus audit)
- `06 Execution/reports/backed-depth-prediction-failure-analysis-2026-07-07-v1.md` (confidence-template + contradiction fixtures)
