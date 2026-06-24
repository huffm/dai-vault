# Supported Sport Availability UX Investigation v1

**date:** 2026-06-24
**status:** INVESTIGATION -- working as designed, NO code change. Frontend trace only; no backend/decision/calibration
change. Records why football does not appear in the analyzer sport selector so it is not re-investigated.
**classification:** product/UX finding. Ran through dai-slice-runner + dai-skill-router gate + systematic-debugging +
product-ui-design-architect + verification-before-completion.

## Question

Why does the analyzer sport selector show only Baseball and Basketball, not Football? Is football hidden by a
data/filter defect, by "available games only" filtering, by buyer-readiness gating, or because no games exist?

## Root cause (verified in source -- the authority)

**Football is hidden intentionally because it is not buyer-ready, not because of game availability.** The selector is
driven by buyer-readiness, never by whether games are currently scheduled.

Evidence chain:

- Backend catalog `platform/dotnet/DevCore.Api/Sports/CompetitionCatalog.cs` is the single source of truth
  (Buyer Packaging Source-of-Truth v1). It sets:
  - NFL: `IsBuyerReady = false`, `ReadinessLevel = smoke_level_only`.
  - NCAAF: `IsBuyerReady = false`, `ReadinessLevel = smoke_level_only`.
  - NBA: `IsBuyerReady = true` (buyer_ready_validated). MLB: `IsBuyerReady = true` (buyer_ready_validated).
  - NCAAMB / College Baseball: not buyer-ready.
  The code comment is explicit: "NFL/NCAAF/NCAAMB are routable but not buyer-ready (smoke-level only)."
- Frontend filter `apps/sports-app/src/app/analyzer/buyer-ready-competitions.ts` keeps only
  `isBuyerReady === true`; `analyzer.component.ts:396` applies it to the fetched competitions before building the
  sport-family options. So the selector = buyer-ready sport families = Baseball (MLB) + Basketball (NBA).
- Existing test `buyer-ready-competitions.spec.ts` already locks this: it asserts the filter "drops competitions the
  platform marks isBuyerReady=false (routable but smoke-only)" using `comp('nfl', false, 'football')`.

So: the selector shows **buyer-ready** sports (not "supported/routable", and not "only sports with available
games"). `isSupported` (routable in code) is a superset of `isBuyerReady`; football is routable-but-smoke-only.

## Answers to the trace questions

- Is the selector showing supported sports? No -- it shows BUYER-READY sports (a deliberate subset of routable).
- Is it showing only sports with available games? No -- it shows buyer-ready sports regardless of current games
  (e.g. NBA shows even when no NBA games are scheduled).
- Is football hidden intentionally, accidentally, or due to no games? **Intentionally** -- NFL/NCAAF are
  `IsBuyerReady=false` / smoke_level_only by platform decision.

## UX contract (current -- already correct)

- Sport selector shows buyer-ready sports. (NBA, MLB) -- correct.
- Game/date area shows available games for the selected sport; when none, an empty state already renders:
  "No upcoming matchups found" (`analyzer.component.ts:222`) / "No games on the books for this pairing" (`:115`).
- Unsupported and internal/smoke-only sports (NFL, NCAAF, NCAAMB) are hidden -- correct buyer-safety.
- No "live now" overclaim. -- correct.

The product already distinguishes buyer-supported sports from currently-available games. Football's absence is
correct, not a bug.

## Decision: no change this slice

No code change. Rationale, per this slice's own authority + buyer-safety rules:

- Making football appear would require flipping backend `IsBuyerReady` for NFL/NCAAF (changing sport-support
  semantics) -- forbidden without a verified defect, and there is none.
- Showing football would violate "do not show internal/dev-only sports" and would overclaim support for a sport
  not validated for buyer presentation (the slice's stated Primary Risk).
- The slice's conditional ("if football is supported but hidden ONLY because no games") is false -- football is
  hidden by the buyer-readiness gate, not by game availability -- so the empty-state UX fix does not apply to it.

## Recommended follow-ups (deferred, optional)

1. **Football Buyer-Readiness Validation (product+platform decision, separate slice):** if/when football is to be
   offered to buyers, deliberately validate it and set NFL `IsBuyerReady = true` / `buyer_ready_validated` in the
   catalog -- then it appears automatically (no frontend change needed) and the existing empty-state covers no-games
   days. This is a product/coverage decision, not a UX bug fix.
2. **No-games empty-state polish (optional UX slice, decoupled from football):** the current empty messages are
   plain text; a future slice could promote them to a compact bordered card for a more intentional feel. Not a
   defect; low priority. Applies to buyer-ready sports without a current slate (e.g. NBA offseason).

## What did not change

No frontend code, no backend sport-support semantics, no decision logic/prompts/confidence/source-depth/advertised
strength/reconciliation/Tool Gateway/calibration data/billing/auth/tenant, no buyer copy. Investigation only.
