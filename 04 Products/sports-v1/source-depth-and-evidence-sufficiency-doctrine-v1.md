# source depth and evidence sufficiency doctrine v1

**status:** active doctrine -- canonical anchor over the existing source/evidence/confidence docs
**date:** 2026-06-23

## purpose

State, in one canonical place, how DAI interprets source depth, evidence sufficiency, confidence, and buyer-facing strength, and how those relate. Several implemented docs already define the pieces; this doc reconciles them into one doctrine so the relationship (not just each piece) is explicit and hard to drift from.

## problem it solves

The pieces are spread across four docs: `confidence-calibration-rules-v1.md` (the two-axis doctrine), `source-group-taxonomy-v1.md` (breadth bands and tiers), `source-depth-contract-v1.md` (depth levels), and `evidence-sufficiency-band-gate-v1.md` (the runtime gate). An agent can read one and miss the relationship -- e.g. treat a high numeric confidence as buyer-facing strength, or conflate breadth with depth. The observed failure that started this line of work was exactly that: `confidence_high_for_partial_evidence` on 4/4 MLB runs (confidence 0.75 on a single grounded signal). This doctrine fixes the relationship so thin support is framed honestly regardless of how coherent the model read is.

## strategic fit

Buyer trust is the protected long-term stock of the factory: it accumulates slowly and drains fast (`confidence-calibration-rules-v1.md`, systems framing). A single thin read advertised as strong withdraws from it. This doctrine is the quality control that keeps the decision product honest about its own support, which is what lets DAI sell repeatable paid decisions rather than one-off lucky-looking calls. It governs product quality, not a feature.

## mental model

Two axes, kept separate, plus a derived buyer surface.

- Axis 1 -- internal coherence (confidence): how coherent/strong the read is from whatever evidence is available. Numeric, relative to the competition's source ceiling, produced by `SportsEvaluator`. Preserved, never mutated by this doctrine.
- Axis 2 -- evidence sufficiency (source support): how much independent grounding actually supports the read. Absolute, derived from `evidenceRichness` (count of grounded signals). Tiered thin / moderate / rich.
- Source support has two sub-dimensions: breadth (how many independent source groups grounded -- `SourceSufficiency`) and depth (how informative each grounded group was -- `SourceDepth`). Breadth and depth are independent; a run can be broad-but-shallow or narrow-but-deep.
- Derived output -- buyer-advertised strength: derived from both axes. Thin evidence caps the advertised band even when confidence is numerically high.

A high model read on thin support is still a thin read. A rich source base does not by itself justify an aggressive claim.

## what it is

- The canonical relationship between source depth, evidence sufficiency, confidence, and buyer strength.
- An anchor that names which existing doc owns each primitive and how they compose.
- A buyer-honesty constraint: thin support cannot present as strong, on first principles, without waiting for outcome data.

## what it is not

- Not a new band/threshold/constant. It does not move the 0.70/0.50 confidence bands, the thin/moderate/rich thresholds, or any `SportsEvaluator` constant.
- Not an outcome-calibration claim. Numeric confidence recalibration stays gated on Outcome Reconciliation (deferred-runtime ledger entry 12).
- Not a runtime change and not a buyer-copy change. It is doctrine over already-implemented behavior.
- Not a replacement for the four anchored docs; it sequences them.

## approved uses

- Deciding how strongly a decision artifact should present itself: read confidence and evidence sufficiency together, never confidence alone.
- Framing source states honestly: missing/`none`, `identity_only`, `shallow`, `enriched` depth, and `proxy`/`fallback_proxy` grounding are described for what they are.
- Choosing buyer copy posture (not facts): a thin or proxy-supported read takes a cautious posture; a rich read may take a firmer (still non-tout) posture.
- Reviewing a slice that touches artifacts, source retrieval, projections, or buyer copy for confidence inflation or false precision.

## disallowed uses

- Advertising "high" buyer strength while evidence sufficiency is thin, regardless of numeric confidence (hard rule from `confidence-calibration-rules-v1.md`).
- Treating numeric confidence as buyer-facing strength.
- Letting rich source support license tout language or unsupported certainty.
- Using evidence sufficiency to mutate facts (lean, posture, confidence value). It guides copy posture only.
- Hiding limitations behind probe-refresh, fallback, or proxy enrichment -- coverage may improve, honesty about remaining gaps may not degrade.
- Exposing internal gap vocabulary to the buyer (`sharp_public`, "missing signal", "fallback failed", "probe required", "source unavailable", "confidence capped") -- see `buyer-copy-safety-v1.md`.

## workflow impact

- Generation/evaluation: confidence is computed as today and preserved; it is provisional until reconciliation.
- Projection: buyer-advertised strength is gated against evidence sufficiency (`gateAdvertisedStrength` in `buyer-signal-summary.ts`): `High` + `thin` caps to `Medium`, records `advertised_strength_limited_by_evidence` (internal, not buyer-rendered), fails open on unknown richness, never raises a band, never mutates confidence/lean/posture.
- Copy: states direction and measured confidence in allowed language; never uses betting-edge posture; never exposes internal gap terms.
- Review: a code-changing slice in this area runs `dai-code-reviewer` and is checked against this doctrine before completion; do not advertise precision the evidence has not earned.

## truth hierarchy

1. Observed runtime behavior and tests (e.g. `buyer-signal-summary.spec.ts`, `SourceSignalTaxonomyTests`, `SourceDepthEvaluatorTests`).
2. Source code (`SportsEvaluator.cs`, `SourceSignalTaxonomy.cs`, `SourceDepth.cs`, `buyer-signal-summary.ts`).
3. Explicit contracts and vault docs (the four anchored docs; this doctrine).
4. Slice handoffs and calibration run notes.
5. Generated graphs (Graphify) and prior assumptions -- navigation only.

When a claim about "how strong is this read" is in play, runtime + tests + source outrank any narrative or graph.

## source or vault references to verify

- `04 Products/sports-v1/confidence-calibration-rules-v1.md` -- two-axis doctrine; relative-completeness-masking-absolute-scarcity failure mode; buyer trust as protected stock.
- `04 Products/sports-v1/source-group-taxonomy-v1.md` -- source groups, Critical/Supporting/Contextual tiers, sport-critical group, breadth bands (insufficient/thin/moderate/rich), null-reason codes.
- `04 Products/sports-v1/source-depth-contract-v1.md` -- breadth vs depth; `SourceDepthLevels none|identity_only|shallow|enriched`; `SourceDepthRecord`.
- `04 Products/sports-v1/evidence-sufficiency-band-gate-v1.md` -- runtime gate; `evidenceSufficiency` thin<=1/moderate==2/rich>=3/unknown; `advertised_strength_limited_by_evidence`.
- `04 Products/sports-v1/buyer-copy-safety-v1.md` -- allowed direction language; forbidden tout posture; forbidden internal gap terms.
- source paths (verify, do not assume): `platform/dotnet/DevCore.Api/AgentRuns/SportsEvaluator.cs`, `SourceSignalTaxonomy.cs`, `SourceDepth.cs`; `apps/sports-app/src/app/analyzer/buyer-signal-summary.ts`.

## example usage

MLB run, starter announced, no published season stats: breadth grounds `[starting_pitching]` -> `SourceSufficiency` band `thin`; depth `starting_pitching = identity_only` (`Observed=true`); `evidenceRichness = 1` -> evidence sufficiency `thin`. Analyzer confidence 0.75 -> raw band `High`. Doctrine result: advertised strength capped to `Medium`, humility reason recorded internally, lean preserved, numeric confidence untouched. Buyer copy: "Slight lean toward [team]; available evidence supports the direction" with measured confidence -- never "best bet" or "expected to cover", never "starter season stats unavailable". A priors-only run (`evidenceRichness 0`, confidence -> `Low`) is left uncapped (the gate only caps High).

## agent prompt guidance

- When asked "how strong is this read", report confidence and evidence sufficiency separately, then the derived buyer strength.
- Never raise buyer strength to match a high confidence number on thin support.
- Distinguish breadth (`SourceSufficiency`) from depth (`SourceDepth`); do not let one imply the other.
- Keep enrichment/fallback/proxy honest: say coverage improved without claiming a gap closed that did not.
- For buyer copy, use the allowed direction vocabulary; never tout posture or internal gap terms.

## acceptance criteria

- Confidence, evidence sufficiency, and buyer-advertised strength are treated as three distinct things.
- No run advertises high buyer strength on thin evidence.
- Breadth and depth are reported independently and honestly (including `none`/`identity_only`/`shallow`/`proxy`).
- No fact (lean, posture, confidence value) is mutated to express sufficiency; only copy posture changes.
- Buyer copy carries no tout language and no internal gap terms.
- Every strength claim is verifiable against source/tests, not narrative.

## risks and failure modes

- Confidence inflation: reading numeric confidence as buyer strength -> blocked by the two-axis separation and the band gate.
- False precision: presenting provisional, not-outcome-calibrated numbers as earned precision -> mitigated by the provisionality rule (numbers stay provisional until reconciliation).
- Breadth/depth conflation: a broad-but-shallow run read as well-supported -> mitigated by keeping `SourceSufficiency` and `SourceDepth` independent.
- Hidden limitations: enrichment/proxy masking a real gap -> disallowed; honesty about remaining gaps must not degrade.
- Doctrine drift as bands/levels change -> mitigated by anchoring to the four source docs and re-checking on changes.

## deferred decisions

- Numeric confidence recalibration against reconciled outcomes (ledger entry 12 / Outcome Reconciliation).
- Server-authoritative advertised strength so non-Angular consumers inherit the humility cap (band gate is presentation-layer today; ledger entry 23).
- Depth-aware breadth band (depth currently nests and does not move the band).
- Promotion of the abstract two-axis model from this sports-v1 doc to a niche-independent `02 Platform` doctrine once a second niche needs it.
- Buyer-visible source depth wording; non-MLB depth coverage; the non-structural null-reason codes.

## related docs

- `04 Products/sports-v1/confidence-calibration-rules-v1.md`
- `04 Products/sports-v1/source-group-taxonomy-v1.md`
- `04 Products/sports-v1/source-depth-contract-v1.md`
- `04 Products/sports-v1/evidence-sufficiency-band-gate-v1.md`
- `04 Products/sports-v1/buyer-copy-safety-v1.md`
- `02 Platform/architecture/sources/decision-freshness-architecture-v1.md` and `04 Products/sports-v1/close-market-confirmation-read-v1.md` -- related but distinct: freshness/close-market describe whether the read is current and market-supported, not how deep the source base is.
- `06 Execution/agent-slice-workflow-doctrine-v1.md` -- the lifecycle this slice followed.

## recommended next slice

Tool Gateway and Agent Permissions Doctrine v1 (next P1 in the backlog). Defer `DAI Slice Runner Skill v1` until the doctrine set is broader. A useful product follow-up, separate from the doctrine backlog, is Advertised-Strength Server Authority v1 (lift the humility cap into the artifact contract so non-Angular consumers inherit it), but only when a second consumer exists.
