---
title: "Handoff -- WI-0005 Identity-Safe Starter Cache v1 (2026-07-11)"
type: "handoff"
date: "2026-07-11"
status: "complete"
project: "DAI"
slice: "WI-0005 Identity-Safe Starter Cache v1"
repos:
  dai: "code (branch, local, not pushed)"
  dai-vault: "docs-only"
tags:
  - handoff
  - sports
  - cache
  - work-item
related:
  - "02 Platform/system-development/work-items/WI-0005-starter-retrieval-caches-transport-failures.md"
  - "06 Execution/reports/v2-accelerated-capture-cadence-wrap-2026-07-11-v1.md"
---

# handoff -- WI-0005 identity-safe starter cache v1

## 1. status

**COMPLETE.** WI-0005 implemented, tested, reviewed, documented, committed **locally on a branch**.
Nothing pushed, no PR (not authorized this slice). Runtime never started (deterministic tests only).

## 2. skills used

- `dai-skill-router` -- gate; selected the set below.
- `dai-slice-runner` -- lifecycle.
- `dai-grill-with-vault` -- reconciled the prompt's "identity-safe cache" framing against the code
  and found a cache already existed at the seam; re-scoped to correct it (not add a second).
- `superpowers:test-driven-development` -- red (compile gap + poisoning repro) -> green -> rest.
- `dai-test-discipline` -- Starter filter first, then full `DevCore.Api.Tests`.
- `superpowers:verification-before-completion` -- every claim tied to a command result.
- `dai-code-reviewer` -- final diff review (APPROVE, 0 blockers).
- `dai-docs-architect` / `dai-agent-handoff` -- this doc + WI + current-slice + MOC.
- Not selected: `systematic-debugging` (defect already root-caused, no live failure to chase).
- Named in the prompt but not loadable as skills: none newly (cohort/pre-settlement discipline are
  patterns, not relevant here).

## 3. repository state before

- dai: `main` @ `bb10c3c` == origin, 0/0; only the stat-only `DevCore.Data.csproj` phantom.
- dai-vault: `main` @ `35ecaf0` == origin, 0/0; two intentional untracked files present.
- runtime: 5007/8000/4200/4201/1433 free, 0 containers (verified).

## 4. work item

- path: `02 Platform/system-development/work-items/WI-0005-starter-retrieval-caches-transport-failures.md`
  (filename kept to preserve the MOC wikilink; title updated to "Identity-Safe Starter Cache v1").
- created 2026-07-10 BACKLOG; this slice moved it to `in-progress` before code, then `complete`.
- scope: one retrieval seam (MlbStarterClient starter acquisition incl. pitcher sub-fetch);
  admission, key, freshness, cancellation; deterministic tests. Out: distributed/db cache, schema,
  background refresh, negative-result caching, stale fallback, doubleheader disambiguation, any
  locked-layer or operational change, WI-0004.
- changes to the WI during execution: title/status; added the as-built implementation summary,
  the doubleheader known-limitation, and the completed links block.

## 5. baseline and root cause

- retrieval path: `RetrieveSignalHandlers (pitching.mlb.probable_starters)` -> `MlbStarterClient.
  GetStartersAsync` -> `FetchStartersAsync` (statsapi `/schedule?hydrate=probablePitcher`) +
  per-pitcher `/people/{id}/stats`.
- repeated-call / poisoning risk established (not assumed): live on 2026-07-10, 15 rapid screens
  produced 5 `network failure` log lines cached as negatives; retries were cache hits; API restart
  + paced re-screen returned all as eligible. 6/15 (40%) false-negative, incl. the rank-1 game.
- seam selected: `MlbStarterClient` (the source-retrieval boundary that owns starter acquisition).

## 6. architecture decision

- placement: at the retrieval seam, not a higher orchestration layer.
- key: `starters:v1:{provider=statsapi}:{competition=mlb}:{home}:{away}:{date}`, inputs trimmed +
  lowercased. Tenant excluded (statsapi starter data is public and invariant). gamePk is NOT a
  lookup key -- it is the call's output, unavailable at lookup, and the retrieval cannot distinguish
  doubleheader games regardless (`FirstOrDefault`). The key is as identity-safe as the retrieval
  contract permits; the residual doubleheader ambiguity is inherited, not introduced, and documented.
- value: only a fully-grounded `MlbStarterGrounding` (`Context != null`); pitcher-quality cache only
  `DataAvailable == true`.
- ttl/freshness: `StarterCacheOptions.Ttl` (config `StarterCache`), default 30m, non-positive falls
  back to default via `EffectiveTtl`. Absolute expiry.
- failure behavior: no stale fallback; after expiry the next call re-fetches. Failures/not-announced
  never cached. `OperationCanceledException` rethrown.
- concurrency: single-flight deferred (documented); `IMemoryCache` is thread-safe; concurrent
  same-key misses may each call the source (bounded, non-corrupting, pre-existing). Correctness
  tested, not exact call count.
- why smallest correct: reuses `IMemoryCache` + `IOptions` + `ISystemClock` (all already present),
  no new dependency, no new interface; the core fix is a two-line admission gate plus a key + ttl +
  cancellation hardening at one seam.

## 7. implementation

- new `platform/dotnet/DevCore.Api/Sports/StarterCacheOptions.cs` (`Ttl`, `EffectiveTtl`, `SectionName`).
- `platform/dotnet/DevCore.Api/Sports/MlbStarterClient.cs`: ctor +`IOptions<StarterCacheOptions>`;
  `StarterCacheKey(...)`; admission gates; ttl from `EffectiveTtl`; `OperationCanceledException`
  rethrow in `FetchStartersAsync` and `FetchPitcherQualityAsync`.
- `platform/dotnet/DevCore.Api/Program.cs`: `Configure<StarterCacheOptions>` from config.

## 8. tests and build

- `dotnet test DevCore.Api.Tests/DevCore.Api.Tests.csproj --filter "FullyQualifiedName~Starter"`
  -> 36 passed (baseline before change: 4 for MlbStarterClient).
- `dotnet test DevCore.Api.Tests/DevCore.Api.Tests.csproj` -> **1092 passed / 0 failed / 0 skipped**
  (baseline 1080; +12 net new after helper de-dup). Build clean (only pre-existing NU1903/CS0108/
  CS8604 warnings; `TreatWarningsAsErrors=false` except NU1605/SYSLIB0011).

## 9. behavioral proof

- miss: first call invokes source once, returns the result unchanged.
- hit: second same-key call within ttl does not re-invoke source (scheduleCalls==1).
- expiry: after ttl (fake clock advanced) the source is called again; a changed starter is visible.
- within-ttl: a later change is NOT visible until expiry.
- isolation: different teams and different dates do not collide (3 distinct calls -> 3 source calls).
- failure not cached: transport 500, network exception, no-match, not-announced, malformed body --
  each followed by a valid response that reaches the provider.
- cancellation: pre-cancelled token throws `OperationCanceledException`, and a later uncancelled
  call succeeds (nothing was cached).
- ttl config: non-positive ttl falls back to default and still caches.
- concurrency: 8 concurrent same-key calls all succeed with equivalent results.

## 10. guardrail verification

- no paid calls, no capture, no reconciliation, no exclusion writes, no runtime start.
- external starter source faked (routing HttpMessageHandler); no live provider called.
- locked layers untouched (diff scoped to MlbStarterClient + StarterCacheOptions + Program.cs DI +
  tests): no prompt/model/confidence/decision/reconciliation/calibration/gate/buyer/registry/schema/
  angular change; no migration.
- unrelated state preserved: `DevCore.Data.csproj` phantom left unstaged; dai-vault untracked files
  untouched; WI-0002/0003/0004 untouched.

## 11. documentation

- WI-0005 (title/status/impl summary/limitation/links/DoD).
- `06 Execution/handoffs/current-slice.md` appended (13403 -> 13450).
- MOC status -> complete.
- this handoff. Lesson recorded (transport failure != cacheable negative evidence); candidate
  lesson noted (preserve indeterminate states) but NOT promoted to doctrine on one instance.
- no ADR: the fix is a local implementation detail, not reusable doctrine.

## 12. commits and final state

- dai: commit `4693b9d` on branch `wi/0005-starter-cache` (from `bb10c3c`). **NOT pushed.**
- dai-vault: one docs commit on `main`. **NOT pushed.**
- dai `main` unchanged at `bb10c3c`. `DevCore.Data.csproj` phantom still unstaged/dirty.
- runtime: never started; ports free, 0 containers (unchanged from cold).
- **Nothing was pushed. No PR, no merge.**

## 13. deferred work and recommended next step

- **Required follow-up:** none for WI-0005 (definition of done met).
- **Recommended next engineering slice (optional, separate approval):** WI-0004 --
  `stop-platform-api.ps1` matches only the `dotnet.exe` host and exits 0 while `DevCore.Api.exe`
  keeps 5007 bound (false success). Small, bounded, honest-shutdown value. Kept fully separate from
  WI-0005; not started here.
- **Documented candidate follow-up (not a WI yet):** doubleheader gamePk disambiguation through the
  `/source-readiness` caller path.
- WI-0002 / WI-0003 remain BACKLOG, untouched. No new slice is authorized by this handoff.
- Integration of WI-0005 (merge/push of `wi/0005-starter-cache`) awaits explicit operator approval.
