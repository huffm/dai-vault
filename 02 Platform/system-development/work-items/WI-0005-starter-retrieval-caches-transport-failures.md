---
title: "WI-0005 Identity-Safe Starter Cache v1 (starter retrieval caches transport failures)"
type: "plan"
date: "2026-07-10"
status: "complete"
project: "DAI"
slice: "WI-0005 Identity-Safe Starter Cache v1"
repos:
  dai: "code"
  dai-vault: "docs-only"
tags:
  - system-development
  - work-item
  - sports
  - retrieval
  - cache
related:
  - "06 Execution/reports/v2-accelerated-capture-day2-2026-07-10-v1.md"
  - "06 Execution/plans/v2-day1-settlement-day2-capture-slice-2026-07-10-v1.md"
  - "02 Platform/decisions/ADR-0008-source-readiness-preflight-gate.md"
---

# WI-0005 identity-safe starter cache v1

**Status: IN PROGRESS. Implementation AUTHORIZED 2026-07-11** (Identity-Safe Starter Cache v1
execution prompt). Registered 2026-07-10 BACKLOG from a defect reproduced during the day-2
capture screen; unblocked after the v2 cadence wrap completed.

**Framing reconciliation.** The registered title ("caches transport failures as missing
starters") and the authorized title ("Identity-Safe Starter Cache v1") describe one work item
from two angles: the defect is that the existing starter cache admits failures; the fix is a
cache that admits only successful, identity-safe results. A cache already exists at the correct
seam, so this slice **corrects the existing cache's admission and key contract** rather than
adding a new one (per the execution prompt's "re-scope to the smallest demonstrable defect in
the existing cache" instruction). Filename kept stable to preserve the MOC wikilink.

## problem

`MlbStarterClient.GetStartersAsync` caches **every** retrieval result for 30 minutes, including
fail-soft `MlbStarterGrounding(null, null)` returns produced by transport failures. A transient
network error is thus stored, for 30 minutes, as a result indistinguishable from "no starters
announced" -- and the cache key (`mlb-starters:{home}:{away}:{date}`) is keyed only by team names
and date, which the identity-safe-key doctrine forbids.

## root cause (reproduced 2026-07-10, live, read-only)

`MlbStarterClient.cs` (as of `bb10c3c`):

- **admission**: `GetStartersAsync` calls `FetchStartersAsync` and unconditionally
  `cache.Set(cacheKey, result, 30 min)` (lines ~123-125). `FetchStartersAsync` returns
  `MlbStarterGrounding(null, null)` on: network exception (~146), non-success HTTP (~154), empty
  schedule (~161), no games (~168), no team match (~190). All five are cached.
- **key**: `mlb-starters:{home.ToLowerInvariant()}:{away.ToLowerInvariant()}:{gameDate}`
  (~115) -- team names + date only.
- **pitcher sub-fetch**: `GetPitcherQualityAsync` (~264-277) has the same class -- it caches
  `Unavailable(...)` failure results for 30 minutes, keyed `mlb-pitcher-season:{id}:{season}`
  (identity-safe key, but failure admitted).
- **cancellation**: the `catch (Exception)` blocks (~143, ~288) swallow
  `OperationCanceledException` and return a fail-soft null, converting a cancellation into a
  domain result.

Live evidence: screening 15 candidates rapidly produced 5 `identity_unmatched` results, all with
`mlb starters: network failure fetching schedule` in the API log; retries were cache hits on the
poisoned negatives; an API restart (which clears the in-process `IMemoryCache`) + a paced serial
re-screen returned all 6 as `matched / enriched / backed_depth / 9 books / eligible`. **6 of 15
candidates (40%) were false-negative**, including the rank-1 narrowest-gap game (824493).

## why now

The v2 cadence closed 2026-07-11; no further paid evidence is authorized and Gate 4 remains false.
The next approved engineering value is source reliability, not capture volume. The defect can
silently distort a paid cohort's selection.

## intended outcome

Repeated valid starter retrievals for the same source identity, within a bounded freshness window,
reuse a cached result; every non-successful result (transport failure, non-success status,
malformed body, no-match, not-announced, cancellation) is **not** cached, so a later call reaches
the provider again. The downstream contract is unchanged.

## architectural boundary

Cache stays at the source-retrieval seam that owns starter acquisition (`MlbStarterClient`). It is
NOT placed in prompt assembly, model analysis, confidence, decision composition, buyer
presentation, reconciliation, calibration, Angular state, or a platform-wide cache framework.

## design (smallest correct)

1. **Admission invariant (the fix):** cache a starter result only when
   `MlbStarterGrounding.Context is not null` -- i.e. both starters announced and enrichment
   attempted. Context is null on every fail-soft path and on genuine not-announced, so all of them
   stay uncached. For the pitcher sub-fetch (same seam), cache only when
   `MlbPitcherQuality.DataAvailable == true`.
2. **Identity-safe key:** `starters:v1:{provider}:{competition}:{home}:{away}:{date}` with
   normalized (trim+lowercase) team inputs. provider (`statsapi`) and competition (`mlb`) are
   constant within this client but included to make the identity contract explicit and prevent
   cross-adapter reuse of the shared `IMemoryCache`. **Tenant is deliberately excluded**: statsapi
   starter data is public and invariant, not tenant-scoped. **gamePk is not a lookup key**: it is
   the *output* of the call, unavailable at lookup, and the retrieval itself cannot distinguish
   doubleheader games (`games.FirstOrDefault`), so the key is as identity-safe as the retrieval
   contract permits. The residual doubleheader ambiguity is a pre-existing property of the
   retrieval, documented as a known limitation and candidate follow-up (out of scope).
3. **Freshness:** TTL is configurable via `StarterCacheOptions.Ttl` (bound from config section
   `StarterCache`, injected `IOptions<>`), default **30 minutes** (unchanged; announced starters
   are stable, and 30 min bounds staleness so a late scratch becomes observable). Non-positive
   configured values fall back to the default. No hidden hard-coded TTL in business logic.
4. **No stale fallback:** absolute expiration; after TTL the entry is a miss and the provider is
   called again. An expired value is never returned.
5. **Concurrency:** `IMemoryCache` is thread-safe; single-flight is **deferred** (documented).
   Concurrent same-key misses may each call the source -- bounded, non-corrupting, and the
   pre-existing behavior. Tested for correctness under concurrency (all succeed, equivalent
   result, cache ends populated), not for exact call count.
6. **Cancellation:** `OperationCanceledException` is rethrown, not swallowed into a fail-soft null,
   and never cached.
7. **Output invariance:** the returned `MlbStarterGrounding` is identical whether served from cache
   or fetched fresh -- no change to names, ordering, quality, identity, source depth, evidence
   richness, confidence, lean, prose, regime, or buyer output.

## in scope

One retrieval seam (`MlbStarterClient` starter acquisition, including its pitcher sub-fetch);
identity-safe key; bounded configurable freshness; positive-result-only admission; cancellation
correctness; deterministic cache tests; existing DI/options patterns; minimal observability using
existing logging.

## out of scope

Redis/distributed/database caching; schema migrations; background/scheduled refresh; cache-admin
endpoints; broad cache infrastructure; market/schedule/lineup caching; **negative-result caching**;
stale fallback; starter prediction/inference; doubleheader gamePk disambiguation through the caller
path; prompt/confidence/source-depth/regime/attribution/artifact/buyer changes; operational capture
or reconciliation; WI-0004.

## risks (evaluated)

cache-key collision (mitigated: provider+competition+normalized identity; doubleheader limitation
documented); stale starter (mitigated: bounded absolute TTL, no stale fallback); caching a
failed/unknown result (mitigated: the admission invariant is the whole point); hiding a late
starter change (mitigated: TTL bounds visibility); bypassing cancellation (mitigated: rethrow);
changing source provenance (mitigated: identical result object cached and returned); cache stampede
(assessed, single-flight deferred, correctness tested); wall-clock test flakiness (mitigated:
fake clock on the cache, no real sleeps); unnecessary abstraction (mitigated: no new interface,
reuse `IMemoryCache` + `IOptions`).

## acceptance criteria

1. First retrieval for an identity invokes the source once and returns the successful result
   unchanged.
2. A repeat within the freshness window does not re-invoke the source and returns a
   contract-equivalent result.
3. Different external game identities, providers, and competitions do not collide.
4. After expiration the next request re-invokes the source; a changed source result becomes visible.
5. Missing/unknown, failed (exception or non-success status), malformed, and cancelled results are
   not cached; the next request re-invokes the source.
6. Cancellation is respected (propagated), never cached as success or failure.
7. TTL binds from configuration; a non-positive value fails safe to the default.
8. Concurrent same-key requests all succeed with equivalent results and leave the cache populated.
9. Downstream output semantics are unchanged (proven by the unchanged existing tests + diff).
10. Full `DevCore.Api.Tests` suite passes; `DevCore.Api` builds; no locked-layer change.

## test plan (written before implementation)

`platform/dotnet/DevCore.Api.Tests/Sports/MlbStarterCacheTests.cs` (new), url-routing fake handler +
invocation-counting, fake `ISystemClock` on the test `MemoryCache` for deterministic expiry:
miss-then-hit (source called once); identity/provider/competition isolation; expiry re-fetch;
changed-result-visible-after-expiry; not-cached for exception / non-success / malformed / no-match /
not-announced; cancellation propagated + not cached; TTL config bind + non-positive fallback;
concurrency correctness. Existing `MlbStarterClientTests` (4) must stay green (output invariance).

## rollback

Remove the admission gate + key change + options and the cache reverts to prior behavior; no
persisted data, schema, artifact, prompt, or decision contract is affected. Revert = the two commits.

## approval boundary

WI-0005 authorizes only the engineering slice described here. It does NOT authorize paid model
execution, sports capture, reconciliation, calibration tuning, Gate 4 changes, pushing/merging/PR,
or WI-0004 work.

## required lifecycle stages

All eight of [[implementation-lifecycle]]. Feature-class review gate not required (no contract or
doctrine surface; `/source-readiness` DTO unchanged -- the fix is internal to the retrieval seam).

## links (completed at close)

- work item: WI-0005 (ADO: AB#— when wired)
- branch: `wi/0005-starter-cache` (from dai `bb10c3c`)
- pr: — (not authorized this slice)
- commits: dai `4693b9d` (implementation + tests, local only, NOT pushed)
- tests: `platform/dotnet/DevCore.Api.Tests/Sports/MlbStarterCacheTests.cs` (13 new);
  `MlbStarterClientTests.cs` (4 existing, unchanged behavior, output-invariance proof)
- verification notes: `DevCore.Api.Tests` 1080 -> 1092 passed / 0 failed / 0 skipped;
  Starter filter 36 passed; dai-code-reviewer APPROVE (no blockers); see the slice handoff
  `06 Execution/handoffs/wi-0005-starter-cache-handoff-2026-07-11-v1.md`
- docs updated: this WI; `06 Execution/handoffs/current-slice.md`; MOC (status complete)
- lessons: transport failure must not be cached as authoritative negative domain evidence;
  candidate (not doctrine): readiness/retrieval seams should preserve indeterminate states
  rather than collapse failure into a cacheable domain result

## implementation summary (as built)

- `platform/dotnet/DevCore.Api/Sports/StarterCacheOptions.cs` (new): `Ttl` (default 30m),
  `EffectiveTtl` fail-safe fallback for non-positive values, `SectionName = "StarterCache"`.
- `platform/dotnet/DevCore.Api/Sports/MlbStarterClient.cs`: ctor takes
  `IOptions<StarterCacheOptions>`; `StarterCacheKey(...)` = `starters:v1:statsapi:mlb:{home}:{away}:{date}`
  normalized; admission gate `if (result.Context is not null)` (starter cache) and
  `if (quality.DataAvailable)` (pitcher cache); TTL from `EffectiveTtl`; `OperationCanceledException`
  rethrown in both fetch methods.
- `platform/dotnet/DevCore.Api/Program.cs`: `Configure<StarterCacheOptions>` bound from config.
- Sole production caller (`Tools/Handlers/RetrieveSignalHandlers.cs`) unaffected: it returns the
  task directly and never relied on cancellation-swallowing; blast radius verified minimal.

## known limitation (candidate follow-up, out of scope)

Doubleheader disambiguation: the retrieval keys by (provider, competition, teams, date) because
gamePk is the call's *output*, not an input, and `FetchStartersAsync` uses `FirstOrDefault` so it
cannot distinguish two same-day same-teams games regardless of the cache. The cache inherits this
limitation; it does not introduce it. Fixing it requires threading a game discriminator through the
`/source-readiness` caller path — a separate WI.

## definition of done -- MET

- [x] acceptance criteria demonstrated by tests (13 new deterministic tests)
- [x] full `DevCore.Api.Tests` suite green (1092/1092)
- [x] code review run (dai-code-reviewer APPROVE, no blockers)
- [x] links block complete
- [x] docs-to-update actioned (WI, current-slice, MOC, handoff)
- [x] lessons recorded
- [x] no locked-layer change; no migration; nothing pushed
