# Stage 0 Larger Identity-Bearing Batch v1

**date:** 2026-06-15
**status:** dry-run / no model spend. prepared an MLB-only candidate batch for later generation + reconciliation; investigated WNBA support and deferred it. no runs generated, no outcomes recorded, no code/schema/prompt/matcher/identity/buyer change. nothing fabricated.
**classification:** sample-preparation + support investigation. NOT a generation or reconciliation slice.

## Why this slice follows the four-candidate reconciliation

Manual Stage 0 Reconciliation Execution v1 (settled, 2026-06-15) reconciled all four prior candidates cleanly through the matcher endpoint (2 correct / 1 incorrect / 1 inconclusive; DB tally 8/8 -> 12/12). That proved the matcher contract end-to-end but on a base too thin to move any confidence/posture threshold (ledger entry 12 stays gated). The recommended next step was MORE Stage 0 samples, preferring non-null leans. This slice prepares that larger batch. Per operator decision, it is **dry-run / no spend**: candidate games are selected and documented, but generation (the first model-spend action) is deferred to an explicitly budgeted slice.

## Pre-state (no spend)

- repos: `dai` clean on `main` 0/0; `dai-vault` on `main` ahead 1 (prior reconciliation commit, unpushed), tree clean.
- services: DevCore.Api up (`GET /api/competitions` 200); `devcore-sql` up (1433 open); FastAPI analyzer down (not needed for a dry-run).
- identity-bearing inventory: 4 runs, **0 unreconciled** -- the four prior candidates are all reconciled, so there is nothing left to *select*; a larger batch must be *generated*.
- dev tenant unchanged: `TenantKey=1`.

## Sport availability (this slice)

| sport | upcoming games | usable this batch | note |
|---|---|---|---|
| MLB | 10 (`/api/competitions/mlb/upcoming?days=3`, all 2026-06-15) | yes | identity provider `mlb_statsapi`; settlement `mlb_statsapi` (same key) |
| NBA | 0 (`/api/competitions/nba/upcoming?days=7`) | no | offseason -- no upcoming games |
| WNBA | n/a -- 404 | no | not supported (see WNBA investigation below) |

Result: the only viable batch is **MLB-only**.

## Prepared MLB candidate batch (pending generation)

These are the upcoming MLB games slated for generation in a future budgeted slice. The upcoming endpoint returns date + teams only; the `mlb_statsapi` gamePk, scheduled UTC start, season, and team refs are captured on the AgentRun row at generation time by `MlbStarterClient` (per MLB Game Identity Capture v1). Lean/posture/confidence/evidence band are unknown until a run is generated -- they cannot be known in advance.

| # | sport | league | teams (away @ home) | date | run id | identity (provider/gamePk) | tenant | lean/posture | directionally usable | evidence band | confidence | settled? | settlement source | readiness |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 1 | baseball | MLB | Miami Marlins @ Philadelphia Phillies | 2026-06-15 | pending gen | mlb_statsapi / pending | 1 | pending gen | unknown until gen | pending | pending | pending | mlb_statsapi gamePk | pending generation |
| 2 | baseball | MLB | Kansas City Royals @ Washington Nationals | 2026-06-15 | pending gen | mlb_statsapi / pending | 1 | pending gen | unknown until gen | pending | pending | pending | mlb_statsapi gamePk | pending generation |
| 3 | baseball | MLB | New York Mets @ Cincinnati Reds | 2026-06-15 | pending gen | mlb_statsapi / pending | 1 | pending gen | unknown until gen | pending | pending | pending | mlb_statsapi gamePk | pending generation |
| 4 | baseball | MLB | San Diego Padres @ St. Louis Cardinals | 2026-06-15 | pending gen | mlb_statsapi / pending | 1 | pending gen | unknown until gen | pending | pending | pending | mlb_statsapi gamePk | pending generation |
| 5 | baseball | MLB | Colorado Rockies @ Chicago Cubs | 2026-06-15 | pending gen | mlb_statsapi / pending | 1 | pending gen | unknown until gen | pending | pending | pending | mlb_statsapi gamePk | pending generation |
| 6 | baseball | MLB | Minnesota Twins @ Texas Rangers | 2026-06-15 | pending gen | mlb_statsapi / pending | 1 | pending gen | unknown until gen | pending | pending | pending | mlb_statsapi gamePk | pending generation |
| 7 | baseball | MLB | Detroit Tigers @ Houston Astros | 2026-06-15 | pending gen | mlb_statsapi / pending | 1 | pending gen | unknown until gen | pending | pending | pending | mlb_statsapi gamePk | pending generation |
| 8 | baseball | MLB | Los Angeles Angels @ Arizona Diamondbacks | 2026-06-15 | pending gen | mlb_statsapi / pending | 1 | pending gen | unknown until gen | pending | pending | pending | mlb_statsapi gamePk | pending generation |
| 9 | baseball | MLB | Pittsburgh Pirates @ Athletics | 2026-06-15 | pending gen | mlb_statsapi / pending | 1 | pending gen | unknown until gen | pending | pending | pending | mlb_statsapi gamePk | pending generation |
| 10 | baseball | MLB | Tampa Bay Rays @ Los Angeles Dodgers | 2026-06-15 | pending gen | mlb_statsapi / pending | 1 | pending gen | unknown until gen | pending | pending | pending | mlb_statsapi gamePk | pending generation |

A future generation slice should generate a capped subset (~8-10, the harness hard cap is 10) via `run-artifact-calibration.ps1 -Competition mlb`, then keep the runs whose `LeanSide` is non-null (directional read) as reconciliation candidates and treat null-lean runs as plumbing-only.

## Identity coverage summary

- existing identity-bearing runs: 4 (all reconciled).
- new identity-bearing runs this slice: 0 (dry-run / no spend).
- prepared (pending-generation) candidates: 10 MLB.
- identity is expected to be captured at generation time (MLB Game Identity Capture v1 is shipped and verified by the four prior MLB candidates that all carried `mlb_statsapi` + gamePk). No identity is captured until a run is generated.

## Counts

- directionally usable candidates (confirmed): 0 this slice (leans not known until generation).
- null-lean / inconclusive candidates: unknown until generation; the prior batch saw 1 of 4 null-lean (Yankees/Blue Jays), so expect a minority.
- pending-settlement: all 10 prepared candidates would be pending settlement (games on 2026-06-15, generated pre-game in a future slice).
- missing-identity: 0 expected (MLB capture is reliable); confirm on the AgentRun row after generation.

## Sport / source breakdown

- MLB: 10 prepared; identity `mlb_statsapi` gamePk; settlement `mlb_statsapi` (same provider/key grounds both run identity and final score).
- NBA: 0 (offseason).
- WNBA: 0 (unsupported).

## Settlement-source note

- **MLB settles through StatsAPI** -- the run's identity provider (`mlb_statsapi` gamePk) is also the settlement source; one key resolves both. Clean, single-provider.
- **NBA / odds-sourced sports require a cross-provider settlement map** -- the run identity is an opaque `odds_api` event id that cannot resolve a final score from a public keyless source; the prior NBA candidate was settled manually from ESPN. Any Stage 1 settlement automation needs a per-sport source map (odds key -> match -> separate box-score provider). Not built here; requirement documented only.

## WNBA support investigation (no spend)

Checked whether WNBA is currently supported, before considering it for the batch. Findings, each evidence-backed:

| # | check | result | evidence |
|---|---|---|---|
| 1 | competition catalog | **absent** | `SportsSeedData.cs` seeds nfl/nba/mlb/ncaaf/ncaamb only; `GET /api/competitions` returns no wnba |
| 2 | odds provider upcoming | **none** | no WNBA sport-key mapping; `GET /api/competitions/wnba/upcoming` -> 404 |
| 3 | analyzer can generate | **no** | FastAPI `_SUPPORTED_COMPETITIONS = {nfl,ncaaf,nba,ncaamb,mlb}` (`services/agent-service/app/routes/sports.py:13`); wnba returns 400 |
| 4 | identity capture | **moot** | no run is created (gated at 400); also no WNBA team seed and no odds sport-key, so even forced it could not resolve teams or an `odds_api` event id |
| 5 | settlement source | **available** | ESPN WNBA scoreboard (`site.api.espn.com/.../basketball/wnba/scoreboard`) returns 200 |

The only "wnba" string in code is a hypothetical in the buyer-ready filter spec (`buyer-ready-competitions.spec.ts:40`, "a future buyer-ready sport (not nba/mlb)") -- a metadata-driven-filter test, not actual support. This empirically confirms Deferred Runtime Decisions Ledger entry 26 ("WNBA... no code path / analyzer / source / seed at all -- Deferred").

**WNBA verdict: defer.** Support is missing at every upstream layer (catalog, team seed, odds sport-key map, analyzer gate). Only the downstream settlement source (ESPN) exists, which is irrelevant while nothing can generate a run. Enabling WNBA is a code + seed slice (new competition + team seed + odds sport-key mapping + buyer-readiness/stable-identity review -- entry 26 flags women's sports as the highest naming/identity risk), which is outside this sample-building slice's boundaries.

## Is any candidate ready for immediate reconciliation?

No. Zero runs were generated (dry-run / no spend), and the prepared games are upcoming. Nothing is reconcilable this slice.

## Recommended follow-up

1. **Generate the MLB batch (budgeted spend slice):** generate ~8-10 of the prepared upcoming MLB games via `run-artifact-calibration.ps1 -Competition mlb`, verify `SourceProvider`/`ExternalGameId` on each AgentRun row, keep the non-null-lean runs as candidates.
2. **Wait for settlement, then reconcile** the non-null-lean candidates through `POST /api/agent-runs/reconcile` with statsapi finals (the proven MLB path).
3. **Do not** build the internal calibration read surface yet -- the sample is still small.
4. **WNBA stays deferred** until a dedicated support slice (catalog + seed + odds map + analyzer gate + readiness review). Settlement (ESPN) is the one piece already in place.
5. **Stage 1 settlement-source map** (cross-provider, for odds-sourced sports) remains the first design item when automation is considered -- not now.

## Decision discipline

The larger Stage 0 batch is *prepared* (MLB-only, pending generation). No candidates are generated or reconciled. The sample base is not yet sufficient for confidence-threshold changes (entry 12 still gated). No model accuracy claim, no buyer-visible track record, no production-readiness claim, no automated feedback loop.
