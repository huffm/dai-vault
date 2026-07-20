---
title: "Market-Contrast Source Adapter Slice 2 Closeout v1 (2026-07-20)"
type: "evidence-report"
date: "2026-07-20"
status: "complete"
project: "DAI"
slice: "WI-0035 Slice 2: bounded default-off mlb source adapter + canonical screen bundle"
repos:
  dai: "code+tests (batch adapter + bounded client refactors; branch wi/0035-market-contrast-source-adapter)"
  dai-vault: "docs-only"
tags:
  - system-development
  - evidence-operations
  - sports-v1
  - implementation
related:
  - "02 Platform/system-development/work-items/WI-0035-market-contrast-candidate-screen.md"
  - "04 Products/sports-v1/market-contrast-candidate-screen-v1.md"
  - "06 Execution/reports/market-contrast-screen-slice-1-closeout-2026-07-19-v1.md"
---

# market-contrast source adapter slice 2 closeout v1

## purpose

Record WI-0035 Slice 2: the bounded, default-off batch source adapter that turns an
explicit frozen MLB candidate slate into classifier facts, results, typed planner
envelopes, and one canonical screen bundle. Committed locally; NOT pushed / NOT merged.
**No live schedule, odds, source-readiness, database, screening, model, generation,
capture, or settlement call occurred anywhere in this slice** -- every behavior is proven
through fake HTTP handlers, an injected fixed clock, and fake readers.

## architecture (dai `a05b63b`, WI: WI-0035)

- `platform/dotnet/DevCore.Api/AgentRuns/MarketContrastSourceAdapter.cs` (new): process-
  scoped default-off gate (`MarketContrastSourceGate`; no endpoint, no DI registration, no
  `.env`/appsettings enablement, no request-body substitute -- a future live authorization
  enables one process and restores default-off); versioned profile
  `market-contrast-source/1.0` with hard server-side limits (MLB only; one date; max 20
  candidates; ONE odds request; `h2h,spreads` x one `us` region = expected 2 credits;
  zero retries; 5-minute quote freshness); orchestration; canonical bundle
  (`market-contrast-screen-bundle/1.0`); the default tenant-scoped batch active-run
  reader (`ExclusionReason == null`, zero writes); atomic bundle publication
  (writer-unique exclusive-create staging + fsync + independent parse + byte compare +
  single atomic replace; no-overwrite admission is an exclusive create).
- bounded refactors, no second authority: `MlbStarterClient.GetStartersForDateBatchAsync`
  (ONE schedule/hydrate request per slate; per-candidate resolution via the extracted
  shared `GroundFromResolutionAsync`, byte-preserving generation semantics; pitcher-stat
  reads deduped by the existing (pitcher, season) cache -- at most two unique reads per
  candidate); `OddsMarketClient.BuildBookReadings` exposed internal with the stable
  `BookmakerKey` (existing behavior unchanged); one small `MarketContrastScreen.JsonString`
  passthrough so the bundle reuses the single escaping authority.

## call budgets (official cost basis recorded)

Odds API `/odds` returns the date's events in one response; quota cost = markets x
regions; `h2h,spreads` x one region = expected 2 credits; `x-requests-last` /
`x-requests-used` / `x-requests-remaining` are the audit headers, recorded in the bundle
without ever recording the API key. Missing, malformed, or greater-than-authorized usage
fails the operation closed after the single call; no second call can occur (tested).
StatsAPI: one schedule/hydrate request per slate + deduped pitcher reads. Database: one
tenant-scoped batch active-run read, zero writes. Zero model calls.

## identity join (fail-closed)

Planner/settlement identity stays `mlb_statsapi`/gamePk; odds provenance additionally
carries `odds_api/<event id>`. Join: each supplied gamePk resolves against the one
StatsAPI date response (resolver semantics, refuses to guess); market matches require
normalized provider team refs in EXACT orientation AND the scheduled-start instant to
match exactly one odds event. Zero or multiple matches are stable blockers
(`no_market_match` / `duplicate_market_match`); doubleheaders join only via their unique
start instants; no team-only, array-position, game-number, gamePk-order, fuzzy, or
orientation-swap matching exists. Join evidence and the odds event id are preserved in
the bundle and envelope provenance.

## freshness policy

`market-contrast-source/1.0`: a book contributes to paired-h2h facts only with a valid
UTC provider update timestamp, not in the future, and age <= 5 minutes at the screen's
as-of. Basis: the provider's official pre-match featured-market update interval is 60
seconds; five minutes is a conservative five-update allowance; repository inspection
found nothing contradicting this basis. The existing generic 15-minute per-match cache is
deliberately not reused as freshness proof. Readiness classification sees the full
observation (source-regime richness); the screen's depth facts use ONLY the fresh
deduplicated paired population -- a stale book can satisfy neither count nor medians.
Timestamps recorded separately: operation started-at, response received-at, per-book
provider updated-at, readiness checked-at, active-run checked-at, screen as-of (stamped
after all evidence, so the screen never predates what it consumes).

## source-failure semantics (recorded honestly)

When the odds source fails (transport/500/429) or no unique market match exists, the
classifier's integrated blocker order reports `source_readiness_not_target_regime`
(envelope status `invalid`, never grounding); the ROOT cause is preserved in the bundle's
source-call ledger and per-candidate join evidence (`source_failed` /
`no_market_match` / `duplicate_market_match`). Stale-only markets surface as
`insufficient_h2h_book_depth` because readiness sees the observation while the screen
sees zero fresh pairs.

## verification

Adapter tests 17/17 (one-request budgets incl. a 15-candidate slate; markets/region
lockdown; non-widenable input contract proven; default-off zero calls incl. zero reader
calls; no-retry on transport/500/429; usage-header contract; 21-candidate rejection
before calls; pitcher dedupe; exact + doubleheader joins; ambiguous/mismatched/wrong-pk/
source-failure blockers; stale quotes non-contributing; thin fresh depth blocking;
active-run exists/not-evaluated blockers; bundle completeness with all candidates incl.
failures, no API key anywhere, all-false authority, not-a-baseline statement; byte
order-invariance + repeatability across fresh adapters; envelope parse + contract fields;
atomic publication incl. two-writer race with exactly one admitted). Envelope-to-python
compatibility is carried by the six byte-identical cross-language vectors and the python
planner suite (same projector). FULL suites: DevCore.Api.Tests **1362 / 0**;
agent-service **560 / 0**; zero new build warnings (pre-existing NU1903 only);
`git diff --check` clean.

## next paid authorization packet requirements (prepared, NOT executed)

A future one-bounded-screen authorization must name: the exact MLB date and candidate
gamePks; the paid source (Odds API `/odds`); max one request; two-credit ceiling; zero
retries; the default-off enable/restore procedure (`EnableForThisProcess` -> run ->
`RestoreDefaultOff`); frozen artifact destination; freshness window
(market-contrast-source/1.0); stop conditions (usage-header violation, join failure);
output schema (`market-contrast-screen-bundle/1.0`); and it must still authorize NO model
generation and NO capture.

## next step

Independent review + integration of the local `wi/0035-market-contrast-source-adapter`
branches (both repos). Only after that, and only under a separately explicit paid-source
approval, may one bounded 2026-07-22 MLB screen and planner pass-2 replay be executed.
A recommendation is not an authorization.

## independent operational-integrity review corrections (2026-07-20 r3, superseding addendum)

A separately authorized review reproduced and corrected, in a new commit (dai `8e044a4`;
originals preserved):

1. **superseded claim -- "default-off process gate":** one `EnableForThisProcess` call
   admitted UNLIMITED sequential and concurrent runs (reproduced: 4 completed runs, 4
   odds calls, accumulated instance counters). replaced by an atomic single-use run lease
   (one admission = at most one source operation; losers make zero source/db calls;
   per-operation ledgers).
2. **superseded claim -- "one odds request per slate":** the odds call was not paid-last
   (reproduced: odds called with an empty schedule and zero resolvable candidates). the
   corrected order is validate -> claim destination -> authoritative statsapi -> readiness
   -> active-run read -> free pre-elimination -> no-paid-call terminal bundle when zero
   candidates remain screenable.
3. **authoritative facts:** caller-supplied teams/start/state were eligibility facts; they
   are now blocking cross-checks against the one statsapi response, and the tenant key was
   removed from the untrusted slate (constructor-bound).
4. **superseded claim -- "generation-readiness parity":** the adapter fabricated a
   BaseballMarketContext with empty spread strings; h2h-only data reached
   `backed_depth`/target regime (reproduced) although the generation path would return no
   market context. readiness now flows through the extracted single normalization
   `OddsMarketClient.TryNormalizeSpread`.
5. **superseded claim -- "eastern date bracket":** `BaseUtcOffset` put july at -05:00.
   the dst-aware `EasternDayBracket` is now the single bracketing authority (july -04:00 /
   january -05:00, both endpoints tested); bounded no-retry timeouts and the exact
   schedule/pitcher/db/odds ledgers (incl. failed-pitcher operation memoization) were
   added; all three quota headers are validated as finite nonnegative numerics with exact
   received values preserved.
6. **source-failure semantics:** classifier 1.2 puts market-observation blockers before
   readiness-regime blockers, so a real outage projects `unavailable` instead of a
   derived readiness reason (cross-language vectors regenerated in both suites).
7. **audited publication:** execution and publication are one operation; every paid
   attempt yields a terminal attempt bundle (incl. quota violations, transport failures,
   malformed responses); staging is writer-unique per attempt (pid alone was
   insufficient); a failed final publication preserves the writer-owned recovery
   artifact; the authority ledger carries booleans only (the not-a-baseline note moved
   out). bundle schema `market-contrast-screen-bundle/1.1`, adapter
   `market-contrast-source/1.1`.
8. **pass-2 replay seam:** planner cli 2.3 gained the deterministic `replay` operation
   (pass-1 request + reviewed bundle -> canonical pass-2 request; strict validation, hash
   preservation, reorder determinism, cross-date/cross-request rejection; no
   hand-composed capability evidence).
9. **executable one-shot operator surface:** `market-contrast-screen` process command
   mode (Program.cs dispatch before the web host; host DI, secrets, db context; one
   attempt; structured status; finally restores default-off; structurally separate free
   `--preflight` whose code path cannot reach the odds call site; no http endpoint).

verification after corrections: adapter 18/18; DevCore.Api.Tests **1363/0**;
agent-service **562/0**; the only two build warnings are the NU1903 package advisory
proven pre-existing (zero csproj changes on the branch); cross-process board sha
`2403c51079eb84cc423acc364c94b1af35ad96a6d14244abf1da4e0ca36fc315`; all six vectors
byte-identical across suites.
