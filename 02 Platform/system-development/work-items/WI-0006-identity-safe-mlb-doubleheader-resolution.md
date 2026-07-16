---
title: "WI-0006 Identity-Safe MLB Doubleheader Resolution v1"
type: "plan"
date: "2026-07-13"
status: "complete"
project: "DAI"
slice: "WI-0006 Identity-Safe MLB Doubleheader Resolution v1"
repos:
  dai: "code"
  dai-vault: "docs-only"
tags:
  - system-development
  - work-item
  - sports
  - retrieval
  - identity
related:
  - "02 Platform/system-development/work-items/WI-0005-starter-retrieval-caches-transport-failures.md"
  - "02 Platform/decisions/0008-source-readiness-preflight-gate.md"
  - "06 Execution/reconciliations/v2-day2-cohort-settlement-2026-07-11-v1.md"
---

# WI-0006 identity-safe mlb doubleheader resolution v1

**Status: IN PROGRESS. Authorized 2026-07-13** (Identity-Safe MLB Doubleheader Resolution v1
execution prompt). Registered as the named follow-up that [[WI-0005-starter-retrieval-caches-transport-failures]]
explicitly deferred: "Fixing it requires threading a game discriminator through the
`/source-readiness` caller path -- a separate WI."

## problem

A team-and-date lookup may correspond to multiple MLB games during a doubleheader. Without a
stable event discriminator, the system can select or cache starter evidence for the wrong game.

`/source-readiness` accepts only `(competition, homeTeam, awayTeam, gameDate)`. That tuple is a
**matchup**, not an event identity. On a doubleheader date it names two distinct games, and every
layer beneath the endpoint silently collapses them into one.

## architectural distinction

> This is an event-resolution problem upstream of cache admission. The cache cannot preserve
> identity that the retrieval contract never received.

WI-0005 made the cache admit only successful, identity-scoped results. It could not make the cache
identity-*correct*, because `gamePk` is the retrieval's **output**, not its input. WI-0006 corrects
the retrieval contract itself; the cache key change is a consequence, not the fix.

## root cause (traced 2026-07-13 against `e8050a9`)

Identity flow, endpoint to provider:

```
GET /source-readiness?competition&homeTeam&awayTeam&gameDate   <- no game discriminator exists
  AgentRunsController.SourceReadiness (~914)
  -> new CompetitionMatchupInput(competition, home, away, date)
  -> new SportsRunArtifact(input, gameDate, throwaway run id)     <- ephemeral, never persisted
  -> SportsRetriever.RetrieveAsync (~36)
     -> ToolGateway: MlbProbableStartersInput(home, away, gameDate)
        -> MlbStarterClient.GetStartersAsync (~129)
           cache key: starters:v1:statsapi:mlb:{home}:{away}:{date}   <- DEFECT 3
           -> GET /api/v1/schedule?sportId=1&date={date}&hydrate=probablePitcher
           -> games.FirstOrDefault(team-name match)                   <- DEFECT 2
           -> GameIdentityContext(gamePk, ...)                        <- gamePk FIRST EXISTS HERE
```

Three defects, one cause:

1. **The contract cannot express intent.** `/source-readiness` has no `gamePk` parameter, so a
   caller who knows which game it wants has no way to say so.
2. **Selection is provider-order-dependent.** `MlbStarterClient.FetchStartersAsync` (~209) does
   `games.FirstOrDefault(g => MatchTeamName(...))` over every game on the date. Two doubleheader
   games both satisfy the predicate; **whichever the provider serialized first wins**, silently,
   with no ambiguity signal. `MatchTeamName` is a bidirectional `Contains` match, so it cannot
   discriminate either.
3. **The cache cannot isolate the games.** Both games normalize to the identical
   `starters:v1:...:{home}:{away}:{date}` key, so Game 1's starters can be served for Game 2.

**Which layer first has a stable `gamePk`:** `MlbStarterClient.FetchStartersAsync`, at the moment
of the schedule match. No caller possesses one earlier. Persisted run/market identities are written
*downstream* of retrieval, so they cannot supply it either. `MlbStarterClient` already owns schedule
resolution as part of its contract (it fetches `/api/v1/schedule` and selects the game); it is not
merely where the cache happens to live.

## reproduction (live, non-paid, read-only, 2026-07-13)

`GET https://statsapi.mlb.com/api/v1/schedule?sportId=1&date=2026-07-11&hydrate=probablePitcher`
returns 16 games, of which **MIL @ PIT is a split doubleheader**:

| | gamePk | start (UTC) | gameNumber | doubleHeader | home probable | away probable |
|---|---|---|---|---|---|---|
| Game 1 | **823357** | 16:05Z | 1 | S | Braxton Ashcraft | Brandon Sproat |
| Game 2 | **823356** | 20:05Z | 2 | S | Bubba Chandler | Shane Drohan |

Three facts this fixture proves:

- The two games have **disjoint starters**. Cross-game contamination is materially wrong evidence,
  not a cosmetic identity error -- the whole starter dimension would be attributed to the wrong game.
- **`gamePk` is not monotonic with game number** (Game 1 has the *higher* pk). Game order cannot be
  inferred from the identifier, and must not be.
- Provider array order currently happens to be `[game 1, game 2]`, so `FirstOrDefault` today returns
  **823357**. Nothing in the provider contract guarantees that order, and nothing in our code depends
  on a guarantee -- it simply takes whatever came first.

**This fixture already cost a paid run.** Per
[[v2-day2-cohort-settlement-2026-07-11-v1]], gamePk 823357 was postponed on 2026-07-10 and
rescheduled into this split doubleheader; captured run `6c9d433e` evaluated the 07-10 event that
never occurred and was **excluded as a postponed non-event**. Postponement is how doubleheaders are
*born*, which puts the postponed/resumed hazard and the doubleheader hazard on the same code path.

Valid defect conditions met: multiple candidates selected without a discriminator; provider order
controls selection; a Game 1 cache entry is reusable for Game 2; the endpoint cannot express caller
intent; an ambiguous matchup is represented as a unique match.

## intended outcome

Every starter-readiness result corresponds to exactly one unambiguous MLB event identity. The system
either resolves exactly one intended `gamePk`, or fails closed with an explicit **ambiguous** result.
It never guesses between same-day duplicate matchups.

## architecture decision

**Selected: Option B (accept optional `gamePk`, fail closed when team/date is ambiguous), with
event resolution owned by `MlbStarterClient`.**

Behavior:

- **`gamePk` supplied** -> resolve that exact event from the schedule; validate it exists on the
  requested date and that its teams match the supplied matchup; retrieve its starters; cache under
  the stable identity.
- **`gamePk` absent, exactly one candidate** -> proceed with its `gamePk` (ordinary behavior, fully
  backward compatible).
- **`gamePk` absent, zero candidates** -> existing truthful no-match behavior, uncached.
- **`gamePk` absent, multiple candidates** -> **explicit ambiguity**. No selection, no starter
  detail retrieval, no cache admission. The response carries the candidate `gamePk`s (plus start
  time and game number) so the caller can retry with one.

Why the others were rejected:

- **Option A (require `gamePk`)** -- rejected. It is the strongest identity, but no current caller
  possesses a `gamePk` before retrieval (it is retrieval's output), so requiring it would break every
  existing caller and the dev tooling for the ordinary single-game case, which is ~99% of traffic.
  Option B reaches the same identity strength on the doubleheader path without that break.
- **Option C (resolve by schedule metadata: game number / start time)** -- rejected **as a selection
  rule**. `gameNumber` and `doubleHeader` are stable and are surfaced as *diagnostics*, but selecting
  by them requires the caller to have expressed intent in the same terms, and no caller does. Picking
  "game 1" for a caller who said nothing about game number is still guessing.
- **Option D (push resolution into `MlbStarterClient`)** -- **accepted on its merits, not by
  default.** The prompt's guard is "do not put orchestration into the client merely because the cache
  is located there." That guard does not bite: the client *already* fetches the schedule and selects
  the game. Resolution is existing client behavior; WI-0006 makes it correct and explicit rather than
  relocating it. Event resolution stays a distinct, separately-testable step from starter parsing.

Where the requested `gamePk` enters: **`SportsRunArtifact`**, not `CompetitionMatchupInput`.
`CompetitionMatchupInput` is both the persisted `AgentRun.InputJson` **and** the public
`CreateAgentRunRequest` body; adding a field there would expand the capture contract and the
persisted payload for a fix that only `/source-readiness` needs. `SportsRunArtifact` is the ephemeral
pipeline carrier that `/source-readiness` constructs directly. **No persisted schema or DB migration
is touched.**

## cache evolution

The lookup happens before `gamePk` is known, so one key cannot serve both phases. Two keys, each
keyed by what it actually is:

- **event resolution:** `mlb-event:v1:statsapi:mlb:{home}:{away}:{date}` -> resolved `gamePk`.
  Admitted **only when exactly one candidate exists**. Ambiguous and zero-candidate resolutions are
  never admitted, so an ambiguous date can never harden into a cached choice.
- **starter content:** `starters:v2:statsapi:mlb:{gamePk}` -> the grounding. `gamePk` is the
  authoritative identity; team/date is no longer authoritative for content once it is known.

This preserves the ordinary-path cache benefit (a repeat team/date request still makes one schedule
call, not two) while making starter identity structurally correct. Game 1 and Game 2 hold distinct
`v2` keys and cannot collide. `v1` entries are in-memory and expire naturally; no migration
framework, no explicit eviction.

## in scope

`/source-readiness` event-identity flow; schedule candidate resolution; the starter retrieval input
contract; stable `gamePk` propagation; cache-key evolution to `v2`; zero/one/multiple-candidate
behavior; deterministic doubleheader fixtures; live non-paid doubleheader verification; precise
ambiguity diagnostics.

## out of scope

Broad endpoint redesign; all-sports identity abstraction; a generic `SportsEventIdentityResolver`
(only one sport path needs it); market snapshot changes; capture orchestration; prompt, decision,
confidence, reconciliation, calibration, Gate 4, buyer surfaces, Angular; persisted schema changes;
selection by unsupported heuristics; using starter names to infer identity; using provider ordering
as an identity rule; using "first game"/"second game" without stable provider evidence; changes to
WI-0004 or WI-0005 beyond the necessary caller-contract update.

## risks (evaluated)

| risk | mitigation |
|---|---|
| assigning Game 1 starters to Game 2 | the defect being fixed; proven absent by disjoint-starter fixtures |
| cross-game cache collision | `v2` key is `gamePk`; distinct entries proven by a cache-isolation test |
| changing behavior for ordinary single games | exactly-one-candidate path is contract-equivalent; existing tests must stay green unmodified |
| ambiguous caller intent | fail closed with candidates; never select |
| provider schedule-order instability | reverse-order test asserts identical behavior; no first-item dependency remains |
| timezone / date-boundary mistakes | the date is passed to the provider as given; no new date math introduced |
| postponed / resumed-game confusion | a resumed game is a distinct `gamePk`; requesting one never silently substitutes another. Same-teams/date collisions now surface as ambiguity rather than a silent pick |
| doubleheader metadata inconsistency | `gameNumber`/`doubleHeader` are reported as diagnostics only, never used to select |
| cache-version migration | in-memory only; `v1` expires naturally |
| overloading `MlbStarterClient` | it already owned schedule resolution; resolution stays separate from starter parsing |
| a new optional identifier callers silently omit | omission is safe by construction: absent `gamePk` + ambiguity = fail closed, never a silent pick |
| misleading "not ready" instead of "ambiguous" | `IdentityStatus: "ambiguous"` is a distinct state with its own stop reason; not collapsed into missing data |
| **generation can no longer analyze a doubleheader** | **accepted, and the point.** It previously analyzed a coin-flip game with possibly-wrong starters. It now degrades to priors-only with unmatched identity. Restoring targeted doubleheader capture requires plumbing `gamePk` into the run input -- deferred, listed below |

## acceptance criteria

1. A single-game team/date request resolves the correct `gamePk` and remains contract-equivalent.
2. An explicit `gamePk` retrieves that exact event; the returned identity matches the request.
3. A doubleheader team/date request with no `gamePk` returns an explicit ambiguous result, selects
   nothing, and admits nothing to the starter cache.
4. Explicit Game 1 `gamePk` returns Game 1 starters; Game 2 data is not used.
5. Explicit Game 2 `gamePk` returns Game 2 starters; Game 1 data is not used.
6. Game 1 and Game 2 occupy distinct cache entries; no cross-game hit.
7. Reversing provider candidate order changes nothing.
8. A `gamePk` whose teams do not match the request fails closed and admits nothing.
9. Zero candidates preserve the truthful no-match behavior.
10. A postponed/resumed event is never silently substituted for the requested identity.
11. Cancellation propagates through every new path and is never cached.
12. Provider failure preserves the existing fail-soft contract and stays uncached.
13. A repeated exact-game request avoids duplicate source work.
14. The public response distinguishes ready / not found / not announced / source failure / ambiguous.
15. Every production caller is updated or proven compatible.
16. Full `DevCore.Api.Tests` green; `DevCore.Api` builds; no locked-layer change; no migration.

## test plan (written BEFORE implementation)

- `platform/dotnet/DevCore.Api.Tests/Sports/MlbDoubleheaderResolutionTests.cs` (new): the 15 cases
  above, driven by the WI-0005 fixture idiom (url-routing `StatefulHandler`, call counting, fake
  `ISystemClock`). Deterministic fixtures modelled on the real 823357/823356 split doubleheader;
  no live provider, no real sleeps, no wall-clock dependency.
- `MlbStarterCacheTests.cs` (13 existing) and `MlbStarterClientTests.cs` (4 existing): must stay
  green **unmodified** -- they are the ordinary-single-game compatibility proof.
- `SourceReadinessClassifierTests.cs`: extend for the `ambiguous` identity status.
- `AgentRunsControllerTests.cs`: `/source-readiness` contract -- optional `gamePk` bind, ambiguous
  response shape.

## verification commands

- `dotnet build platform/dotnet/DevCore.Api/DevCore.Api.csproj` -> succeeded, no new warnings
- `dotnet test platform/dotnet/DevCore.Api.Tests/DevCore.Api.Tests.csproj` -> **1120 passed /
  0 failed / 0 skipped** (1092 at WI-0005 close; +28)
- live, non-paid: `GET /health`; `GET /source-readiness` for ordinary single game, ambiguous
  doubleheader, game 1, game 2, cache-isolation repeat, identity mismatch against a warm cache,
  and `gamePk=0` -> 400. Re-run in **adversarial order** (explicit gamePk before the ambiguous
  request) after the B2 fix below.

## two blocking bugs found in verification (both fixed, regression-tested)

Recorded because each was caught by a *different* gate -- neither was sufficient alone.

- **B1, found by LIVE verification while unit tests were green.** The gamePk cache fast-path
  returned a warm entry without re-validating the matchup, so `gamePk=823357` requested under
  Yankees/Red Sox was served Pittsburgh's starters. Unit tests missed it because each starts
  with a cold cache. Fixed: `MlbEventResolver.IdentityMatchesMatchup` re-checks every cache hit.
- **B2, found by CODE REVIEW while live had passed.** The matchup->gamePk resolution entry was
  written unconditionally, *including when an explicit gamePk was supplied*: resolving game 1
  explicitly taught the cache that "PIT/MIL/07-11" means game 1, so the next ambiguous request
  would have been silently answered with it -- the removed guess re-entering through the cache.
  Live missed it only because the first pass asked the ambiguous question *before* the explicit
  ones. Fixed: write the resolution entry only when `requestedGamePk is null`.

## approval boundary

WI-0006 authorizes identity-safe source-readiness resolution only. It does not authorize paid
execution, sports capture, reconciliation, calibration changes, prompt changes, decision changes, or
unrelated backlog work. Local commits only: no push, merge, PR, rebase, amend, or branch deletion.

## links

- work item: WI-0006 (ADO: AB#— when wired)
- branch: `wi/0006-doubleheader-gamepk-resolution` (from dai `e8050a9`) — **pushed to origin
  2026-07-13; retained**
- pr: — (merged direct via fast-forward: `e8050a9..4f8f381`)
- commits: dai `4f8f381` (implementation + tests) — **integrated to dai/main and pushed**
  2026-07-13 (clean fast-forward, no merge commit; main tree == branch tree; dai/main ==
  origin/main at `4f8f381`); dai-vault `0db3cd2` (WI closeout + handoff) + integration addendum
- tests: `platform/dotnet/DevCore.Api.Tests/Sports/MlbDoubleheaderResolutionTests.cs` (20 new);
  `SourceReadinessClassifierTests.cs` (+4); `Integration/AgentRunsControllerTests.cs` (+4);
  `MlbStarterCacheTests.cs` (13) + `MlbStarterClientTests.cs` (4) pass **unmodified** — the
  ordinary-single-game compatibility proof
- verification notes: `DevCore.Api.Tests` 1092 -> **1120 passed / 0 failed / 0 skipped**;
  `DevCore.Api` builds; live-verified against the real 2026-07-11 MIL@PIT split doubleheader;
  dai-code-reviewer **APPROVE**; see
  `06 Execution/handoffs/wi-0006-doubleheader-resolution-handoff-2026-07-13-v1.md`
- docs updated: this WI; `06 Execution/handoffs/current-slice.md`; MOC (WI-0006 registered);
  the WI-0006 handoff
- lessons:
  1. **A cache keyed by identity can still launder a wrong identity.** Keying content by `gamePk`
     is necessary but not sufficient — the *request* must still be validated against the entry, or
     the fast path becomes a bypass of the check the cold path performs (B1).
  2. **A resolution cache must only record what was actually resolved.** Writing a matchup ->
     identity entry after an *explicitly identified* call teaches the cache a guess it never made
     (B2). Cache what you concluded, not what you were told.
  3. **Cold-start unit tests cannot see cache-order defects.** Both bugs were invisible to a suite
     where every test constructs a fresh cache. Live verification and adversarial-order replay are
     not ceremony here; they are the only gates that could see these.
  4. Candidate (not doctrine): retrieval seams should make *under-specified request* a distinct
     outcome from *missing data*. They fail in opposite directions — one is fixed by asking a better
     question, the other by waiting for the source.

## integration record (2026-07-13)

Re-verified before integration, all evidence regenerated (not inherited from the build slice):
invariant greps against `4f8f381` (no `FirstOrDefault` selection, v2 gamePk key, resolution entry
gated on `requestedGamePk is null`, warm-hit revalidation, `gamePk<=0` -> 400, no migration);
**B1 and B2 regressions run by name** and green; focused 67; full suite **1120/1120**; build clean;
live scenarios A–H including **Scenario G in both adversarial orders** (explicit G1-first and
explicit G2-first, ambiguity survived both); warm-cache mismatch failed closed; `gamePk=0` -> 400.
Accepted limitation proven structurally: `AgentRunService` constructs the artifact with no gamePk,
so ambiguous doubleheaders degrade to priors-only/unmatched in generation. Final review: APPROVE,
zero blockers. Integrated by fast-forward only; feature branch pushed and retained.

## deferred (NOT authorized by this WI -- two separate items, do not conflate)

1. **Propagate `gamePk` through `CompetitionMatchupInput`** so capture/generation can target a
   specific doubleheader game. Until then, generation fails closed (priors-only, unmatched
   identity) on an ambiguous doubleheader rather than guessing. Contract-expansion work: touches
   the persisted `InputJson` and the public analyze request body, so it needs its own WI and review.
2. **Evaluate first-class `no_match` / `ambiguous` / `source_failure` identity statuses** on
   `/source-readiness`. Today `ambiguous` is first-class but `no_match` and `source_failure` share
   the `unmatched` status, distinguishable only via `IdentityReason`. Diagnostic-contract work,
   independent of item 1.

Also deferred: any cross-sport event-identity abstraction.
