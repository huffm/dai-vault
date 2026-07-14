---
title: "WI-0012 Settled Outcome Recap v1 -- slice handoff (2026-07-14)"
type: "handoff"
date: "2026-07-14"
status: "complete"
project: "DAI"
slice: "WI-0012 Settled Outcome Recap v1"
repos:
  dai: "code (recap projection + export + endpoints; local branch, not pushed)"
  dai-vault: "docs (WI, MOC, current-slice, this handoff; local)"
related:
  - "02 Platform/system-development/work-items/WI-0012-settled-outcome-recap.md"
  - "02 Platform/system-development/work-items/WI-0011-buyer-decision-brief-contract.md"
  - "04 Products/sports-v1/v1-release-definition-and-scope-freeze-v1.md"
---

# wi-0012 slice handoff -- settled outcome recap v1

**Disposition: IMPLEMENTATION COMPLETE, LOCAL ONLY. Review resolved. Nothing pushed.**

Governing WI: WI-0012 (`complete`; full validation record inside the WI). Commits: dai
`7152818` on `wi/0012-settled-outcome-recap` (from `140b5a2`; 5 files, +989/-43);
dai-vault docs commit on main (this handoff, WI, MOC, current-slice entry).

## what shipped

The canonical buyer postgame recap -- the concierge follow-up deliverable:

1. **`BuyerSettledRecap.cs`** -- `BuyerSettledRecapDto` (closed state vocabulary:
   not_settled / settled_evaluated / settled_not_evaluated / no_position / excluded /
   no_result / inconsistent; server-composed ResultLabel + ResultNote; outcome block with
   final scores + winning team + settled-at; evaluation block rendering the PERSISTED
   verdict verbatim with a fixed buyer-safe explanation) embedding the WI-0011
   `BuyerDecisionBriefDto` verbatim as the original read, + `BuyerRecapMarkdownRenderer`
   (deterministic, invariant-culture, claim-safe).
2. **Endpoints** `GET /api/agent-runs/{id}/recap` and `/recap/markdown`, same auth/404
   semantics as /brief; ONE consolidated loader now serves all four buyer surfaces
   (/brief endpoints return the recap's embedded Read), with the claim-safety
   suppression tripwire and disambiguated fail-closed warnings on that single path.
3. **Fail-closed honesty rules:** non-final settlements (cancelled/postponed/suspended/
   void/unknown) -> `no_result`, never "evaluation pending"; evaluation-without-outcome
   and unknown eval statuses -> `inconsistent` + diagnostic warning, never an invented
   result; no-position reads show the score but are NEVER scored; excluded runs render
   exactly "No result — event not evaluated."; partial score residue omitted from json
   and markdown consistently. Per-read only -- no aggregate record anywhere.

## verification

DevCore.Api.Tests **1212/1212** (baseline 1176, +36); sports-app vitest **134/134**
(regression; no Angular production change). Live GET-only on real runs: 823845 settled
Correct w/ byte-identical markdown; 823357 excluded; a real lean-null run -> no_position
(score shown, not scored); a real outcome-less run -> not_settled. Runtime cold at start
and end. Focused review (3 angles): 9 findings fixed, 1 refuted -- detail in the WI.

## deferred (not authorized, not started)

1. WI-0012 integration and push.
2. WI-0013 pilot operations hardening (duplicate guard, metering, runbook, RC drill).

### Slice Synopsis

**Change:** Shipped the canonical settled-outcome recap (projection + deterministic
markdown export on /recap[/markdown]) embedding the WI-0011 brief, with a fail-closed
honest state machine, and consolidated all four buyer surfaces onto one loader.
**Reason:** Second V1 critical-path item -- the product promises a settled outcome after
completion, and no buyer surface carried one.
**Proof:** Red-first 1212/1212 C# + 134/134 vitest; live checks on the real settled,
excluded, no-position, and unsettled runs; review 9/10 fixed, 1 refuted with doctrine.
**State:** dai `7152818` local on `wi/0012-settled-outcome-recap`; vault docs committed
locally; nothing pushed; runtime cold; 0 paid calls, 0 captures, 0 DB writes.
**Next:** Separately authorize WI-0012 integration and push, then WI-0013.
