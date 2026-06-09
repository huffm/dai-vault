# Readiness-Gated Buyer Packaging v1

**date:** 2026-06-09
**status:** buyer-packaging safety. narrow Angular buyer-copy + selector-gating changes + tests in `dai`; report/handoff/ledger in `dai-vault`. no source/prompt/parser/schema/artifact-contract/buyer-projection-semantics/confidence/posture/lean/signal/model/cost change; no new sport support.
**scope:** ensure buyer-facing packaging does not imply broader sports coverage than the Pre-Support Buyer Validation Matrix v1 supports.

## Purpose

Technical generation is not buyer support. The Pre-Support Buyer Validation Matrix v1 found only NBA and MLB are buyer-ready validated; NFL, CFB (ncaaf), and NCAAB men (ncaamb) are smoke-only; WNBA, NHL, and NCAAB women are unbuilt. This slice checks whether any buyer-facing surface implies support beyond NBA/MLB and corrects it narrowly.

Primary question: does any frontend text, selector, label, example, or support wording imply sports beyond NBA and MLB are buyer-ready? Answer: yes -- several did; corrected.

## Scope

- inspect the Angular sports app buyer-facing surfaces (landing, analyzer, account, history, header) and the dev surface.
- narrowly align buyer packaging with the readiness matrix; prefer hiding/demoting/caveating smoke-only sports over expanding support.
- add targeted tests; no runtime artifact-semantics change.

## Non-goals

- no new sport support, no source integration, no prompt/parser/schema/artifact-contract change.
- no buyer-projection semantics, confidence/posture/lean/signal/model/cost change; no probe-refresh; no outcome reconciliation.
- no pricing/Stripe/auth/tenant/dashboard/deployment change. no Jera change. no push.

## Documents / code inspected

- `app.routes.ts` (buyer vs dev routes), `sports-api.service.ts` (`getCompetitions`), `core/models/agent-run.model.ts` (`CompetitionReferenceDto`).
- buyer-facing: `landing/landing.component.ts` + `.html`, `analyzer/analyzer.component.ts` (sport selector derivation), `account/account.component.html`, `history/history.component.ts`.
- dev: `dev-artifact-review/*` (the `/dev/artifacts` route).
- doctrine: Pre-Support Buyer Validation Matrix v1; backend `CompetitionCatalog` (the `isSupported` flag).

## Buyer-facing surface inventory

- buyer-facing routes: `/` (landing), `/analyzer` (the matchup analyzer + buyer signal summary), `/history`, `/account`.
- dev/internal route: `/dev/artifacts` (dev artifact review; a raw competition dropdown -- acceptable as an internal surface, but the route is unguarded).
- sport/competition exposure: the analyzer sport selector is populated from `/api/competitions`; the landing has a `coverage[]` strip; account has a plan-description line; history renders hardcoded mock reads.
- the catalog's `isSupported` flag means "routable in code," NOT "buyer-ready" -- the core confusion this slice corrects.

## Support-claim findings (before)

Multiple buyer-facing surfaces overclaimed support, several nearly inverted vs the matrix:

- landing `coverage[]`: NFL = "Live Now" (false -- smoke-only); MLB = "Planned" (false -- MLB is buyer-ready validated, the strongest current surface); NCAAF/NCAAB = "Next" (overstated). Only NBA "Live Now" was correct.
- landing hero sample: an NFL matchup ("Baltimore @ Cincinnati") badged "Live Now," implying NFL is a live product.
- analyzer selector: exposed all routable competitions (NFL, NCAAF, NBA, NCAAMB, MLB) as selectable, letting a buyer run smoke-only reads (it gated on `isSupported` = routable, not buyer-ready).
- account plan line: "NFL and NBA live now" (NFL false; omits the actually-validated MLB).
- history mock reads: 2 of 5 sample rows were NFL, implying NFL reads are produced.

WNBA, NHL, and NCAAB women were not mentioned anywhere (correct).

## Buyer-copy safety findings

- no lock/guarantee/must-bet/hammer language was found in the surfaces changed.
- the buyer-projection layer (`buyer-signal-summary`) and the evidence-sufficiency band gate are untouched -- buyer artifact meaning is unchanged.
- residual (pre-existing, not a support overclaim): the 3 remaining history mock rows use betting-price leans ("-5.5", "-145"); this is demo tone, recorded as a follow-up, not fixed here to keep the change narrow and avoid altering more copy than necessary.

## Changes made

All narrow, frontend-only, no runtime artifact semantics:

1. analyzer selector gated to buyer-ready sports: new pure module `analyzer/buyer-ready-competitions.ts` (`BUYER_READY_COMPETITION_CODES = ['nba','mlb']` + `filterBuyerReadyCompetitions`); `analyzer.component.ts` applies it to the `getCompetitions()` result, so the buyer selector only offers NBA and MLB. Smoke-only competitions remain runnable internally via the calibration harness / direct API -- the platform is unchanged, only the buyer UI is gated.
2. landing `coverage[]` realigned to the matrix: NBA and MLB are "Live Now" (validated, measured-support notes); NFL is "Next" (internal validation target); NCAAF/NCAAB are "Planned" (seasonal validation targets). Comment records that statuses are gated by the matrix, not code-path existence.
3. landing hero sample switched from the NFL matchup to an NBA matchup ("Boston @ New York", "NY -2.5 to -3.5") so the most prominent "Live Now" sample is a buyer-ready sport.
4. account plan line: "NFL and NBA live now" -> "NBA and MLB live now".
5. history mock reads: the 2 NFL rows replaced with an NBA read (measured Medium, rest+market grounded) and an MLB read (split/no-clear-lean), so buyer-facing history shows only supported sports.
6. tests: `buyer-ready-competitions.spec.ts` (5 tests: keeps nba/mlb, drops nfl/ncaaf/ncaamb, drops null-code, empty-safe, allowlist is exactly nba+mlb) and `landing.component.spec.ts` (3 tests: only NBA/MLB are Live Now, no smoke-only sport is Live Now, deferred sports absent) -- written test-first (RED on the missing module + the inverted coverage data), then green.

## Product posture

- what the product can safely say now: NBA and MLB are buyer-ready ("Live Now"), framed as measured decision support; support varies by slate and signal availability.
- what should remain hidden/internal: NFL, CFB (ncaaf), NCAAB men (ncaamb) -- routable and surfaced only as future/seasonal validation targets and on the `/dev/artifacts` dev surface; never as live buyer support.
- what should be deferred: WNBA, NHL, NCAAB women -- not mentioned to buyers.
- a "supported sports" source of truth: today the buyer-ready set is a frontend allowlist (`BUYER_READY_COMPETITION_CODES`). A future backend `isBuyerReady` flag (distinct from `isSupported`) or readiness config should drive packaging so the matrix controls the UI from one place. Deferred -- recorded against ledger entry 26.
- packaging should eventually be driven from readiness metadata, not duplicated allowlists.

## Risks and gaps

- the buyer-ready allowlist is duplicated conceptually in two places (the frontend filter and the landing coverage list + its test); a single backend `isBuyerReady` source of truth would remove the drift risk.
- `/dev/artifacts` is an unguarded route exposing the full competition dropdown; acceptable as an internal surface but a route guard / build-time exclusion is recommended hardening (not this slice).
- history is hardcoded mock data ("no backend wiring in v1"); its remaining betting-price lean tone is a copy-polish follow-up.

## Recommended next work

- backend `isBuyerReady` flag on the competition reference (or a readiness config), with the frontend packaging and selector reading it -- removes the duplicated allowlist.
- a dev-route guard for `/dev/artifacts`.
- when football/college seasons return, Football Pre-Support Validation v1 can promote NFL/CFB by adding their codes to the readiness source of truth after validation.
- a small Buyer Copy Polish pass on the history mock data tone.

## Recommended next slice

Outcome Reconciliation Runtime v1 (sequenced next). The readiness source-of-truth (`isBuyerReady`) and the `/dev/artifacts` guard are small hardening follow-ups that can be folded into a later packaging slice; neither blocks the reconciliation runtime.

## What was not changed

- no new sport support, source integration, prompt, parser, schema, or artifact contract
- no buyer-projection semantics or buyer artifact meaning (buyer-signal-summary + band gate untouched)
- no confidence/posture/lean/signal/model/cost change; no probe-refresh; no outcome reconciliation; no provider choice
- no pricing/Stripe/auth/tenant/dashboard/deployment change; no Jera change
- backend competition catalog and `/api/competitions` unchanged (gating is frontend-only)

## Verification

- Vitest: 46 tests pass (5 files; +8 new), no regressions. `ng build`: bundle generation complete, exit 0.
- buyer-surface sweep: only the landing future-target entries (NFL "Next", NCAAF/NCAAB "Planned") remain, framed as roadmap, never live; the landing spec enforces none are "Live Now".
- `git diff --check` clean; added-line exact-path scan; non-ASCII scan 0 across changed/new code and vault docs.
