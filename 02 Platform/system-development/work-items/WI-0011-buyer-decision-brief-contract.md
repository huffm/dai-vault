---
title: "WI-0011 Buyer Decision Brief Contract v1"
type: "plan"
date: "2026-07-14"
status: "complete"
project: "DAI"
slice: "WI-0011 Buyer Decision Brief Contract v1"
repos:
  dai: "code (buyer projection + export + analyzer panel; branch wi/0011-buyer-brief-contract)"
  dai-vault: "docs (this WI, release-plan correction, current-slice, handoff at close)"
tags:
  - system-development
  - work-item
  - product
  - buyer
related:
  - "04 Products/sports-v1/v1-release-definition-and-scope-freeze-v1.md"
  - "06 Execution/plans/v1-release-critical-path-2026-07-14-v1.md"
  - "04 Products/sports-v1/buyer-copy-safety-v1.md"
  - "04 Products/sports-v1/buyer-artifact-route-v1.md"
---

# wi-0011 buyer decision brief contract v1

**Slice type:** production behavior change (buyer-facing contract + presentation).
**Opened:** 2026-07-14. First implementation item of the frozen V1 critical path.

## problem

The buyer-facing analyzer panel renders the RAW `POST /api/agent-runs` response: a
numeric confidence tile ("75%") and 0.70/0.45-threshold "Strong/Mixed/Weak" labels --
an implied-probability claim the 15-row v2 evidence contradicts (discrimination
inverted). Market context is not rendered; game identity is echoed from Angular form
state (gamePk never displayed); there is no deliverable rendering of the brief. Buyer
semantics are split between C# and Angular (the evidence band lives in a frontend
mapper), so claim-safety is enforced by view discipline, not by contract.

## decision (governing product + architecture)

- The backend owns ONE canonical buyer decision brief projection and the deterministic
  Markdown export produced from it; Angular renders that projection and derives no
  buyer strength language, market wording, identity, or claim-safety decisions.
- Numeric model confidence and any label derived from the raw 0.70/0.45 thresholds are
  removed from every buyer-facing pregame surface; numeric confidence remains on
  internal/dev diagnostic surfaces only.
- The evidence-gated strength band is the sole buyer-facing strength language; raw
  confidence is never renamed into "evidence strength".
- Market context is a deterministic plain-language field derived from persisted market
  data (agreement / disagreement / deliberate divergence / unavailable / unclear),
  claim-safe per `buyer-copy-safety-v1`.
- Identity (competition, teams, gameDate, requested + resolved gamePk, generated-at)
  comes from the persisted run and artifact, never form state.

## scope

1. Canonical buyer brief projection (server-side, extends the existing
   `BuyerArtifactProjection` boundary): identity block, stance/lean + explicit
   no-position state, evidence-gated band, market-context line, buyer-safe prose
   (summary, factors, counter-case, watch-for), source-depth/evidence-availability
   language already approved.
2. Numeric-confidence removal from all buyer surfaces (panel, projection, export,
   secondary buyer cards); threshold labels removed; internal surfaces unchanged.
3. Angular analyzer buyer panel consumes the canonical projection (no POST-response
   rendering, no form-state identity, no frontend-derived buyer semantics).
4. Deterministic Markdown export generated server-side from the same projection,
   exposed on the smallest read-only endpoint; no email sending or delivery automation.

## non-goals / exclusions

WI-0012 (outcome recap); WI-0013 (duplicate guard, metering); real history; email
sending; Stripe; deployment; auth changes; prompt/model changes; confidence
recalibration; scoring/threshold changes; new signals; identity-status refinement;
WI-0002/0003; doubleheader capture; DB migrations; push (integration separately gated).

## acceptance criteria

1. No buyer-facing surface displays numeric or percentage confidence.
2. No buyer-facing surface uses raw-confidence threshold labels.
3. Internal diagnostic surfaces retain required confidence information (test-proven).
4. Buyer panel renders the canonical persisted projection.
5. Identity comes from the persisted run, not form state.
6. Evidence-gated band is the sole buyer-facing strength language.
7. Market context renders safely for agreement, disagreement, divergence, unavailable,
   and unclear cases.
8. Markdown export is deterministic for a fixed run.
9. Panel and export share the same canonical projection.
10. No-position fixture renders an explicit no-position brief.
11. Claim-safety checks cover the projection and export.
12. No prompt, scoring, confidence, routing, reconciliation, settlement, calibration,
    or database-schema behavior changes.

## required fixtures

Ordinary lean + market agreement; ordinary lean + market disagreement; deliberate
divergence naming the market-favored team (823845-shaped); explicit no-position;
unavailable/insufficient market context; attribution-Unclear (822877-shaped); persisted
requested + resolved gamePk identity; claim-unsafe text rejected/sanitized; fixed-run
export determinism; internal surface retaining numeric confidence.

## validation record (2026-07-14)

- red-first throughout: brief types (compile red -> 42 unit + 7 integration green), the
  angular mapper reshape (vitest compile red -> green), and the review-driven regex
  precision fix (8 red -> green).
- suites: DevCore.Api.Tests **1176/1176** (from 1168 pre-review-fixes; 1127 at WI-0009);
  sports-app vitest **134/134** across 13 spec files; angular production bundle compiles.
- live read-only verification on real persisted runs (devcore-sql + DevCore.Api from the
  branch; runtime returned to cold): 823845 (609d433e, the deliberate-divergence row)
  renders identity (gamePk 823845, requestedGamePk null pre-WI-0009), the fidelity-guard
  deliberate-divergence market line naming Miami Marlins (9 books), band High, claim-safe
  prose; markdown export byte-identical across two calls, text/markdown, no run id, no
  "confidence", no numeric model confidence; 822882 (lean-null) renders explicit
  no-position + "books currently lean Detroit Tigers (6 books). This read does not take a
  side."; /artifact/buyer wire has NO confidence key while internal /artifact retains
  confidence + analyzerConfidence.
- /code-review high (8 angles + verify): 10 findings resolved -- copy-safety regex made
  phrase-precise (ordinary baseball vocabulary no longer suppresses sections) + plural
  internal-gap fix; single tiebroken market subquery (tie could mix batches); coarse
  grounded/missing lists added to the brief (legacy runs keep their signal summary);
  invariant-culture date formatting; history page numeric confidence + threshold badge
  removed; landing static "Confidence 71%" chip replaced; live brief-failure now an
  explicit degraded state (POST-prose fallback gated to dev stub mode only); suppression
  tripwire logging; dead EvidenceRichness removed from the buyer wire.
- accepted/deferred (documented, not defects): AgentRunResultDto (create response) still
  carries numeric confidence for operator tooling -- deferral comment on the contract; no
  buyer surface renders it. requestedGamePk stays on the brief (operator contract;
  panel renders resolved pk). stance-label vocabulary exists in C# renderer and TS panel
  (single-sourcing = candidate follow-up). legacy rows with null LeanSide render
  no-position (fail-closed; v2-era buyer runs all carry LeanSide).

## links

- work item: WI-0011
- branch: `wi/0011-buyer-brief-contract` (dai, from `e64567f`)
- pr: -- (not authorized)
- commits: dai `140b5a2` (20 files, +1546/-359; implementation + tests + review fixes;
  "LOCAL ONLY" superseded 2026-07-24 audit -- integrated per final disposition below)
  + dai-vault docs commit at close (this WI, freeze-doc RC criterion 1
  correction, MOC, current-slice, handoff)
- tests: DevCore.Api.Tests 1176/1176; sports-app vitest 134/134
- verification notes: validation record above; live checks on 609d433e / 822882
- docs updated: this WI; freeze-doc RC criterion 1 correction (operator-authorized);
  MOC; current-slice; handoff
- lessons: none recorded at close (explicit none, 2026-07-24 completion audit -- no
  reusable-lesson section was authored for this slice; single-sourcing of read blocks
  was recorded above as a candidate follow-up instead)

## final disposition

Complete + integrated (2026-07-14). Integration commit dai `140b5a2` == dai/main ==
origin/main, a pure fast-forward from `e64567f` (main tree byte-identical to the reviewed
branch tree `b3cbf68`); branch `wi/0011-buyer-brief-contract` pushed and RETAINED local +
remote at `140b5a2` per the branch-retained convention. Pre-integration re-verification on
the reviewed commit: DevCore.Api.Tests 1176/1176, sports-app vitest 134/134, bundle
compiles. Review resolved (10 findings fixed). No implementation work remains open.
This supersedes the prior "implementation complete, local only" disposition.
