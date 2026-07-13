---
title: "WI-0006 Identity-Safe MLB Doubleheader Resolution v1 -- slice handoff (2026-07-13)"
type: "handoff"
date: "2026-07-13"
status: "complete"
project: "DAI"
slice: "WI-0006 Identity-Safe MLB Doubleheader Resolution v1"
repos:
  dai: "code (local branch, not pushed)"
  dai-vault: "docs-only (local, not pushed)"
tags:
  - execution
  - handoff
  - system-development
  - sports
  - retrieval
  - identity
related:
  - "02 Platform/system-development/work-items/WI-0006-identity-safe-mlb-doubleheader-resolution.md"
  - "02 Platform/system-development/work-items/WI-0005-starter-retrieval-caches-transport-failures.md"
  - "06 Execution/reconciliations/v2-day2-cohort-settlement-2026-07-11-v1.md"
---

# WI-0006 slice handoff -- identity-safe mlb doubleheader resolution v1

**Disposition: COMPLETE, LOCAL ONLY. Code review APPROVE. Nothing pushed.**

> A matchup is not an event identity. When more than one game can satisfy the same teams and date,
> the system must resolve a stable provider identity or declare ambiguity. It must never guess.

## 1. starting state (verified before any change)

- dai: `main` @ `e8050a9`, `main == origin/main`, clean except the known
  `DevCore.Data.csproj` phantom (preserved untouched; its content diff vs main is empty).
- dai-vault: `main` @ `1bb15b1`, `main == origin/main`, two intentional untracked files untouched.
- remote WI branches `wi/0004-truthful-api-shutdown` and `wi/0005-starter-cache` retained.
- runtime cold: 5007/8000/4200/4201/1433 free, zero containers, `devcore-sql` stopped, docker
  daemon not even running.

Branch cut after baseline capture: `wi/0006-doubleheader-gamepk-resolution` from `e8050a9`.

## 2. identity flow (traced against e8050a9)

```
GET /source-readiness?competition&homeTeam&awayTeam&gameDate   <- no game discriminator existed
  AgentRunsController.SourceReadiness
  -> CompetitionMatchupInput -> SportsRunArtifact (ephemeral, never persisted)
  -> SportsRetriever.RetrieveAsync
     -> ToolGateway: MlbProbableStartersInput(home, away, gameDate)
        -> MlbStarterClient.GetStartersAsync
           cache key: starters:v1:statsapi:mlb:{home}:{away}:{date}      <- both DH games collide
           -> GET /api/v1/schedule?sportId=1&date&hydrate=probablePitcher
           -> games.FirstOrDefault(team-name match)                      <- provider order decides
           -> GameIdentityContext(gamePk, ...)                           <- gamePk FIRST EXISTS HERE
```

`gamePk` first becomes available **inside `MlbStarterClient` at schedule-match time**. No caller has
one earlier; persisted run/market identities are written downstream of retrieval. `MlbStarterClient`
already owned schedule resolution (it fetched the schedule and picked the game) -- WI-0006 makes that
existing behavior correct and explicit rather than relocating orchestration into it.

## 3. reproduction (live, non-paid, read-only)

`schedule?sportId=1&date=2026-07-11&hydrate=probablePitcher` -- MIL @ PIT is a **split doubleheader**:

| | gamePk | start (UTC) | gameNumber | home probable | away probable |
|---|---|---|---|---|---|
| Game 1 | **823357** | 16:05Z | 1 | Braxton Ashcraft | Brandon Sproat |
| Game 2 | **823356** | 20:05Z | 2 | Bubba Chandler | Shane Drohan |

- starters are **disjoint** -> a cross-game hit is materially wrong evidence, not a cosmetic error.
- `gamePk` is **not monotonic with game number** (game 1 holds the higher pk) -> order is never
  inferable from the identifier.
- `823357` is the game that was **postponed on 2026-07-10 and rescheduled into this doubleheader**;
  run `6c9d433e` evaluated the 07-10 event that never happened and was excluded as a postponed
  non-event (`v2-day2-cohort-settlement-2026-07-11-v1`). Postponement is how doubleheaders are born,
  so the postponed/resumed hazard and the doubleheader hazard sit on the same code path.

## 4. what shipped

| file | change |
|---|---|
| `Sports/MlbEventResolver.cs` **(new)** | event resolution, separate from starter parsing: matched / no-match / **ambiguous** / identity-mismatch. Never selects by provider order, game number, or start time. |
| `Sports/MlbStarterClient.cs` | `FirstOrDefault` guess removed; identity-aware `GetStartersAsync` overload; cache re-keyed to `starters:v2:statsapi:mlb:{gamePk}`; new `mlb-event:v1:...` resolution entry admitted only for an unambiguous matchup. |
| `Sports/GameIdentity.cs` | `MlbEventCandidate`, `MlbEventAmbiguity`, `MlbEventUnresolved`; `MlbStarterGrounding` gains `Ambiguity` + `UnresolvedReason`. |
| `AgentRuns/SportsRunArtifact.cs` | `RequestedGamePk` (ephemeral carrier -- **not** on the persisted `CompetitionMatchupInput`). |
| `AgentRuns/SportsRetriever.cs`, `Tools/Handlers/RetrieveSignalHandlers.cs` | propagate the requested identity. |
| `AgentRuns/SportsRetrievalOutput.cs` | carries ambiguity alongside `GameIdentity` (same never-persisted class of metadata). |
| `AgentRuns/SourceReadiness.cs` | `IdentityStatus: "ambiguous"` + `Candidates` + `IdentityReason`. |
| `Controllers/AgentRunsController.cs` | optional `gamePk` query param (+ non-positive -> 400). |

**Contract:** with `gamePk` -> that exact event, after validating it is the matchup described.
Without -> zero candidates = truthful no-match; one = unchanged ordinary behavior; **many = explicit
ambiguity** with candidate gamePks, no selection, no starter-detail call, no cache admission.

## 5. two blocking bugs -- each caught by a different gate

This is the most transferable lesson of the slice: **neither gate alone was sufficient.**

- **B1 -- found by LIVE verification, unit tests green.** The `gamePk` cache fast-path returned a warm
  entry *without* re-validating the matchup. `gamePk=823357` requested under Yankees/Red Sox was
  served Pittsburgh's starters. Unit tests missed it because each starts with a cold cache.
  Fixed by `MlbEventResolver.IdentityMatchesMatchup` re-checking every cache hit.
- **B2 -- found by CODE REVIEW, live had missed it.** The matchup->gamePk resolution entry was written
  unconditionally, *including when an explicit gamePk was supplied*: resolving game 1 explicitly
  taught the cache that "PIT/MIL/07-11" means game 1, so the next ambiguous request would have been
  silently answered with it -- the removed guess, re-entering through the cache. Live missed it only
  because the first pass happened to ask the ambiguous question *before* the explicit ones.
  Fixed by writing the resolution entry only when `requestedGamePk is null`, then re-verified live in
  **adversarial order** (explicit first, then ambiguous).

## 6. verification

- `dotnet test DevCore.Api.Tests` -> **1120 passed / 0 failed / 0 skipped** (1092 at WI-0005 close; +28).
- The 17 existing WI-0005 starter tests pass **unmodified** -- the ordinary-single-game compatibility
  proof. The 4-arg `GetStartersAsync` signature is byte-identical; identity-awareness is an overload.
- `dotnet build DevCore.Api` -> succeeded, no new warnings.
- Live `/source-readiness` (API on 5007 + `devcore-sql`): ordinary single game (LAA@MIN `823682`)
  unchanged and enriched; ambiguous DH refuses with both candidates; game 1 -> Ashcraft/Sproat;
  game 2 -> Chandler/Drohan; cache isolation holds on repeat; identity mismatch fails closed against a
  **warm** cache; `gamePk=0` -> 400.
- dai-code-reviewer: **APPROVE**, no remaining blockers.

**Locked layers unchanged:** prompts, registry, model routing, confidence, scoring, decision, market
agreement, reconciliation, calibration, Gate 4, buyer output, DB schema, Angular apps. No migration.
The `v1`->`v2` cache-key bump is a direct consequence of stable `gamePk` propagation.

## 7. accepted behavior change (read this before capture resumes)

Generation can **no longer analyze an ambiguous doubleheader**. It degrades to priors-only with
unmatched identity instead of silently analyzing a coin-flip game with possibly-wrong starters. That
is strictly better -- the previous behavior could attribute game 1's starters to game 2 and then
reconcile against the wrong outcome -- but it means a doubleheader is currently **uncapturable** until
`gamePk` is plumbed into `CompetitionMatchupInput`. That is deferred work, not authorized here.

## 8. final state

- dai: `wi/0006-doubleheader-gamepk-resolution`, 1 local commit. **Not pushed.** `main` untouched.
- dai-vault: `main`, 1 local commit (docs). **Not pushed.** Two intentional untracked files untouched.
- runtime cold, independently confirmed: 5007/8000/4200/4201/1433 free, zero containers,
  `devcore-sql` exited, no `DevCore.Api` / `uvicorn` / `dotnet` (MSBuild node-reuse servers also
  shut down).

## 9. deferred (NOT authorized -- do not begin)

- WI-0006 integration and push.
- `gamePk` on `CompetitionMatchupInput` so capture/generation can target a doubleheader game.
- Distinguishing "no match" from "source failure" as separate `IdentityStatus` values (they are
  already distinguishable via `IdentityReason`; the status collapse is pre-existing).
- WI-0002, WI-0003.
- Any cross-sport event-identity abstraction (only one sport path needs it today).
