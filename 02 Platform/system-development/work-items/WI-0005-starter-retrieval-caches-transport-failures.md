---
title: "WI-0005 Starter Retrieval Caches Transport Failures As Missing Starters"
type: "plan"
date: "2026-07-10"
status: "blocked"
project: "DAI"
slice: "WI-0005 starter retrieval caches transport failures"
repos:
  dai: "unchanged"
  dai-vault: "docs-only"
tags:
  - system-development
  - work-item
  - backlog
  - sports
  - retrieval
related:
  - "06 Execution/reports/v2-accelerated-capture-day2-2026-07-10-v1.md"
  - "06 Execution/plans/v2-day1-settlement-day2-capture-slice-2026-07-10-v1.md"
  - "02 Platform/decisions/ADR-0008-source-readiness-preflight-gate.md"
---

# WI-0005 starter retrieval caches transport failures as missing starters

**Status: BACKLOG. Implementation authorization: NOT AUTHORIZED.**
Registered 2026-07-10 from a defect reproduced during the day-2 capture screen. The capture
proceeded only after an operational workaround (API restart); no code was changed.

## problem statement

`MlbStarterClient.GetStartersAsync` fails **soft** on retrieval failure — a transport
exception or a non-success status returns `new MlbStarterGrounding(null, null)`
(`MlbStarterClient.cs` ~138-155). The caller then **caches that null result for 30 minutes**
(`SetAbsoluteExpiration(TimeSpan.FromMinutes(30))`, ~line 125), keyed
`mlb-starters:{home}:{away}:{date}`.

A transient network failure is therefore persisted, for half an hour, as a result
indistinguishable from the legitimate condition "no probable starter announced". Downstream:

- `SourceReadinessClassifier` reads it as `starter=missing` and `identityStatus=unmatched`
- `/source-readiness` reports `generationEligibleForTargetRegime=false`,
  `stopReason="identity unmatched; starter=missing"`
- a screen silently drops the candidate; a **generation** would stamp
  `observedDataRegime=starter_missing_*` instead of the intended backed-depth regime

The failure is invisible at the API surface. Only the server log distinguishes the two:
`mlb starters: network failure fetching schedule for ...` vs
`mlb starters: no game matched ... -- verify team names`.

## current evidence (reproduced 2026-07-10, day-2 capture screen)

Screening 15 candidates in rapid succession produced 5 `identity_unmatched` results plus one
readiness error. Server log for exactly those games:

```
mlb starters: network failure fetching schedule for Baltimore Orioles vs Kansas City Royals on 2026-07-10
mlb starters: network failure fetching schedule for Cincinnati Reds vs Chicago Cubs on 2026-07-10
mlb starters: network failure fetching schedule for Tampa Bay Rays vs Seattle Mariners on 2026-07-10
mlb starters: network failure fetching schedule for Miami Marlins vs Cleveland Guardians on 2026-07-10
mlb starters: network failure fetching schedule for New York Mets vs Boston Red Sox on 2026-07-10
```

Retrying with backoff did **not** clear them — every retry was a cache hit on the poisoned
negative (`mlb starters cache hit: mlb-starters:...`). After restarting `DevCore.Api` (which
clears the in-process `IMemoryCache`) and re-screening **serially with pacing**, all six games
returned `identity=matched, starter=enriched, market=backed_depth, bookCount=9, eligible=true`.

**Impact measured: 6 of 15 candidates (40%) were false-negative.** The narrowest de-vigged gap
game on the slate (824493 CHC@CIN, gap 0.0209) was among them and would have been silently
excluded from a paid capture cohort ranked on that gap.

## affected surfaces

- `platform/dotnet/DevCore.Api/Sports/MlbStarterClient.cs` (fail-soft return + cache write)
- `platform/dotnet/DevCore.Api/Sports/OddsMarketClient.cs` — inspect for the same pattern
  (market also cached; not yet proven to share the defect)
- `GET /api/agent-runs/source-readiness` (consumer; reports the poisoned value as ineligibility)
- Any capture screen or generation run inside the 30-minute TTL

## classification

**Non-blocking defect with a real capture-integrity impact.** It cannot corrupt settled
evidence (a poisoned run lands in a `starter_missing` regime and would simply not be a
backed-depth row), but it can (a) silently distort cohort *selection*, and (b) waste paid model
calls on a run that cannot reach the target regime. It blocked nothing in this slice once the
cache was cleared.

## scope

Make retrieval failure distinguishable from absent data, and stop caching failures as facts.

## non-goals

No change to the readiness classifier's semantics, the regime taxonomy, capture ranking rules,
prompt/model/confidence/decision behavior, or `ADR-0008`'s core rule that readiness must reuse
the generation retrieval path. No new external dependency. No retry storm.

## execution authority

None. Registered, not authorized. Must not be implemented inside a capture or settlement slice.

## activation gate

Implement before the next paid capture cohort that ranks candidates on a screened attribute,
or sooner if any screen again shows an unexplained cluster of `identity_unmatched`.

## decision space (all open)

1. **Do not cache failures.** Cache only successful groundings; on failure return without
   `cache.Set`. Simplest; a failing upstream is then retried each call (bounded by call volume).
2. **Distinguish the outcomes in the type.** Return a discriminated result
   (`Grounded` / `NotAnnounced` / `RetrievalFailed`) so the classifier can emit
   `identityStatus=unknown` + a distinct `stopReason`, and `/source-readiness` can surface
   "screen unreliable" rather than "ineligible". Most faithful; touches the readiness contract,
   so it needs the architecture-contracts lens.
3. **Negative-cache with a short TTL** (e.g. 30s) and a bounded internal retry, keeping the
   30-minute TTL for successes only.

Option 1 is the minimal correct fix; option 2 is the honest one. They compose.

## acceptance criteria (for the future implementation slice)

1. A simulated transport failure is never served from cache on the next call.
2. A genuine "no probable starter announced" is still cached and still reads as
   `starter=missing`.
3. `/source-readiness` cannot report `eligible=false` for a game whose ineligibility was caused
   solely by a retrieval failure — it reports a distinguishable state, or it retries.
4. No change to eligibility semantics for games with genuinely absent starters.
5. Existing readiness tests stay green; `/metrics` byte-identical.

## test plan (written before implementation)

Unit: inject an `HttpClient` handler that throws, then one that succeeds; assert the second
call performs a live fetch (no cache hit) and returns grounded starters. Assert a successful
"no starters announced" response IS cached. Classifier tests: a `RetrievalFailed` grounding
must not map to the same `stopReason` as `starter=missing` (if option 2 is chosen).
Integration: existing `SourceReadiness` tests unchanged and green.

## verification commands

`dotnet test platform/dotnet/DevCore.Api.Tests` (targeted: starter-client + source-readiness
fixtures), plus a live screen of a full slate asserting zero unexplained `identity_unmatched`.

## dependencies

None. Independent of WI-0001..WI-0004.

## risks

- Removing the negative cache increases StatsAPI call volume on a genuinely failing upstream —
  the very condition that caused the failures. Pair option 1 with pacing or a short negative TTL.
- Option 2 changes a read contract (`/source-readiness` response), so it is a proposal requiring
  the architecture-contracts review gate, not an edit.

## rollback / containment posture

Single-client change plus optional readiness-contract addition. Revert = one commit. No data,
schema, settlement, or buyer surface. **Operational containment that works today: restart
`DevCore.Api` to clear the in-process cache, then screen serially with pacing.**

## required lifecycle stages

All eight of [[implementation-lifecycle]]. Feature-class **only if** option 2 is chosen (it
touches a response contract); otherwise lite tier.

## required links (at close)

The standard 8 links per [[work-item-traceability]].

## definition of done

Per [[implementation-lifecycle]], plus acceptance criteria 1-3 demonstrated by tests, and a
full-slate live screen with zero unexplained unmatched rows.

## lesson (already actionable, no code required)

A screen that ranks paid capture candidates must treat a **transport failure as a hard error**,
never as a data verdict. The 2026-07-10 screen was corrected by failing loudly and re-screening;
the frozen slate was rebuilt before any spend. This mirrors ADR-0008's lesson one level down:
readiness must reuse generation retrieval **and** must not silently inherit its fail-soft
behavior as an eligibility signal.
