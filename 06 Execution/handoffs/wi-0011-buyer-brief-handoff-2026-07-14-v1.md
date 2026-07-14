---
title: "WI-0011 Buyer Decision Brief Contract v1 -- slice handoff (2026-07-14)"
type: "handoff"
date: "2026-07-14"
status: "complete"
project: "DAI"
slice: "WI-0011 Buyer Decision Brief Contract v1"
repos:
  dai: "code (buyer projection + export + panel; local branch, not pushed)"
  dai-vault: "docs (WI, freeze-doc RC correction, MOC, current-slice, this handoff; local)"
related:
  - "02 Platform/system-development/work-items/WI-0011-buyer-decision-brief-contract.md"
  - "04 Products/sports-v1/v1-release-definition-and-scope-freeze-v1.md"
  - "06 Execution/plans/v1-release-critical-path-2026-07-14-v1.md"
---

# wi-0011 slice handoff -- buyer decision brief contract v1

**Disposition: IMPLEMENTATION COMPLETE, LOCAL ONLY. Review resolved. Nothing pushed.**

Governing WI: WI-0011 (`complete`; full validation record inside the WI). Commits: dai
`140b5a2` on `wi/0011-buyer-brief-contract` (from `e64567f`; 20 files, +1546/-359);
dai-vault docs commit on main (this handoff, WI, MOC, freeze-doc RC criterion 1
correction, current-slice entry).

## what shipped

One canonical, server-owned buyer pregame brief:

1. **`BuyerDecisionBrief.cs`** -- `BuyerDecisionBriefDto` (persisted identity incl.
   requested + resolved gamePk and generated-at; stance + explicit no-position; the
   evidence-gated `AdvertisedStrength` band as the ONLY strength language; deterministic
   plain-language market context derived from the persisted snapshot + the existing
   `MarketAttributionFidelity` guard; claim-safe prose with fail-closed transport
   suppression via phrase-precise `BuyerCopySafety`; coarse legacy signal lists) +
   `BuyerBriefMarkdownRenderer` (deterministic, invariant-culture, no internal
   identifiers -- the concierge email deliverable).
2. **Endpoints** `GET /api/agent-runs/{id}/brief` and `/brief/markdown` (same tenant +
   user scoping and 404 semantics as the artifact endpoints; single tiebroken market
   subquery; suppression tripwire logging).
3. **Numeric-confidence removal from every buyer surface:** analyzer confidence tile +
   0.70/0.45 "Strong/Mixed/Weak" labels deleted; `BuyerArtifactDto` no longer carries
   confidence on the wire; history page numeric + threshold badge removed (band-only
   "Evidence strength"); landing static "Confidence 71%" chip replaced. Internal
   /artifact, prompt-trace, calibration surfaces unchanged (test-proven).
4. **Angular panel renders the canonical projection:** identity from the brief (never
   form state), market-context section, band-only strength, no-position rendering;
   frontend band derivation deleted (server band verbatim); POST-response prose renders
   only in dev stub mode; live brief-fetch failure shows an explicit degraded notice.

## verification

DevCore.Api.Tests **1176/1176**; sports-app vitest **134/134** (13 files); bundle
compiles. Live read-only checks on real runs 823845 (deliberate-divergence market line,
byte-identical markdown, no confidence on the buyer wire) and 822882 (explicit
no-position + books-lean line). Runtime cold at start and end. `/code-review high`:
10 findings, all fixed (regex precision + plural gap, market subquery tiebreak, legacy
coarse signals, invariant culture, history/landing scrubs, degraded state, suppression
logging, dead EvidenceRichness). Deferred with notes: create-response confidence
(operator tooling; documented on the contract), stance-label single-sourcing.

## deferred (not authorized, not started)

1. WI-0011 integration and push.
2. WI-0012 settled outcome recap; WI-0013 pilot ops hardening.
3. AgentRunResultDto confidence transport removal; stance-label single-sourcing.

### Slice Synopsis

**Change:** Shipped the canonical server-owned buyer decision brief (projection +
deterministic markdown export + endpoints) and removed numeric confidence and threshold
labels from every buyer surface; the panel now renders the persisted projection.
**Reason:** First V1 critical-path item -- the buyer panel rendered a raw "75%" tile and
threshold labels the 15-row inverted-discrimination evidence cannot support, and no
deliverable brief rendering existed.
**Proof:** Red-first tests (1176/1176 C#, 134/134 vitest), live checks on 823845/822882,
review with 10 findings fixed.
**State:** dai `140b5a2` local on `wi/0011-buyer-brief-contract`; vault docs committed
locally; nothing pushed; runtime cold; 0 paid calls, 0 captures, 0 DB writes.
**Next:** Separately authorize WI-0011 integration and push, then WI-0012.
