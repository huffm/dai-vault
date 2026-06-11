# Buyer Packaging Source-of-Truth v1

**date:** 2026-06-09
**status:** source-of-truth / contract hardening. narrow backend catalog+DTO change and narrow frontend consumption change + tests in `dai`; report/handoff/ledger in `dai-vault`. no schema/migration, no artifact-contract change, no sports-support expansion, no source integration, no prompt/parser/buyer-projection/confidence/posture/lean/model/cost change.
**scope:** move buyer-readiness from a frontend-only allowlist to an explicit platform-owned source of truth.

## Purpose

Readiness-Gated Buyer Packaging v1 corrected buyer overclaims but kept the buyer-ready set as a frontend hardcoded allowlist (`['nba','mlb']`). That is a safe stopgap, not a durable source of truth. This slice moves buyer readiness into the platform competition catalog as explicit metadata and has the frontend consume it.

Primary question: where should DAI store and expose buyer-readiness so packaging, selectors, and claims are driven by one explicit contract instead of duplicated frontend knowledge? Answer: the platform `CompetitionCatalog`, exposed on the competitions API, consumed by the frontend.

## Scope

- add explicit buyer-readiness metadata (`IsBuyerReady`, `ReadinessLevel`) to the platform competition catalog (in-code static data; no DB).
- expose it on `GET /api/competitions`.
- replace the frontend allowlist with metadata-driven filtering, fail-safe.
- tests proving routable-in-code is distinct from buyer-ready, and that only NBA/MLB are buyer-ready.

## Non-goals

- no schema/migration, no artifact-contract change, no new sport support, no source integration.
- no prompt/parser/buyer-projection/confidence/posture/lean/signal/model/cost change; no probe-refresh; no outcome reconciliation.
- no pricing/Stripe/auth/tenant/dashboard/deployment change. no Jera change. no push.

## Documents / code inspected

- `platform/dotnet/DevCore.Api/Sports/CompetitionCatalog.cs` (the `CompetitionDefinition` source of truth + `CompetitionCodes`).
- `platform/dotnet/DevCore.Api/Controllers/SportsReferenceController.cs` (`CompetitionReferenceDto` + `GET /api/competitions`, where `IsSupported` is computed).
- `apps/sports-app/src/app/core/models/agent-run.model.ts` (`CompetitionReferenceDto`), `sports-api.service.ts` (`getCompetitions`), `analyzer/analyzer.component.ts` (selector), `analyzer/buyer-ready-competitions.ts` (the prior allowlist).
- doctrine: Pre-Support Buyer Validation Matrix v1; Readiness-Gated Buyer Packaging v1.

## Existing catalog/API behavior (before)

- `CompetitionCatalog.All` is the authoritative in-code list of competitions (`CompetitionDefinition`: code, display, sport family/level, odds key, expected signals -> `MaxGroundedSignals`). It is static data, not a DB table -- so adding metadata needs no migration.
- `GET /api/competitions` (`SportsReferenceController.GetCompetitions`) maps the catalog to `CompetitionReferenceDto` and computes `IsSupported = c.Code is not null && activeSet.Contains(c.Code)` -- i.e. routable + active. There was no buyer-readiness concept; `IsSupported` conflated routability with support.
- the Angular `CompetitionReferenceDto` carried `isSupported` + `availabilityNote`; the analyzer selector previously filtered through a frontend allowlist (`BUYER_READY_COMPETITION_CODES = ['nba','mlb']`).

## Source-of-truth decision

Buyer readiness lives in `CompetitionCatalog` (the platform/catalog layer), not in the frontend and not in a DB. Reasons: the catalog is already the single source of competition truth; it is static code (no migration); and readiness is a small, explicit product fact that belongs next to the competition definition. The frontend consumes it via the existing competitions API. This removes the duplicated frontend allowlist.

The smallest shape that prevents overclaiming: one boolean (`IsBuyerReady`) plus a `ReadinessLevel` string. No `buyerAvailabilityLabel`/`buyerReadinessNote` were added (avoid overbuild).

## Metadata shape

- `CompetitionDefinition.IsBuyerReady: bool` -- buyer packaging may present this competition as currently available. Distinct from `IsSupported`/routability.
- `CompetitionDefinition.ReadinessLevel: string` -- a classification from the new `ReadinessLevels` constants: `buyer_ready_validated`, `buyer_ready_limited`, `smoke_level_only`, `contract_ready_source_limited`, `deferred`, `not_supported`.
- per competition: nba and mlb -> `IsBuyerReady true`, `buyer_ready_validated`; nfl, ncaaf, ncaamb -> `IsBuyerReady false`, `smoke_level_only`; the non-routable College Baseball placeholder -> `IsBuyerReady false`, `not_supported`.
- `CompetitionReferenceDto` gains `IsBuyerReady` + `ReadinessLevel`, serialized camelCase like the existing `isSupported` (confirmed by the existing frontend already consuming `isSupported`).

## Frontend consumption behavior

- the Angular `CompetitionReferenceDto` gains optional `isBuyerReady?: boolean` and `readinessLevel?: string`.
- `filterBuyerReadyCompetitions` now filters on `c.isBuyerReady === true` -- the platform flag -- instead of the code allowlist (which was removed). The analyzer selector's call site is unchanged.
- fail-safe: a competition is shown to buyers only when the platform explicitly says `isBuyerReady === true`; a missing or false flag is hidden. So an older backend (no flag) or a routable-but-smoke-only competition is never presented to buyers.
- dev/internal tooling (`/dev/artifacts`) continues to use routability (`isSupported`) and is unaffected.

## Buyer safety validation

- backend: `CompetitionCatalogReadinessTests` proves nba/mlb are `IsBuyerReady` (`buyer_ready_validated`), nfl/ncaaf/ncaamb are routable-but-not-buyer-ready (`smoke_level_only`), only nba+mlb are buyer-ready, and the non-routable placeholder is not buyer-ready.
- frontend: `buyer-ready-competitions.spec` proves the filter is metadata-driven, not code-driven (a buyer-ready flag on a non-nba/mlb code is honored; nba is dropped if the platform says not buyer-ready), and fails safe (missing flag -> hidden).
- the analyzer buyer selector still shows only NBA/MLB (now because the platform marks only those buyer-ready). NFL/CFB/NCAAB men do not appear as buyer-ready; WNBA/NHL/NCAAB women are absent from the catalog and unaffected.
- landing coverage, account plan line, and history (aligned in the prior slice) are unchanged and remain measured; no new aggressive/touty language.

## Changes made

backend (`dai`):
- `CompetitionCatalog.cs`: added `ReadinessLevels` constants; added `IsBuyerReady` + `ReadinessLevel` to `CompetitionDefinition`; set them on all six catalog entries.
- `SportsReferenceController.cs`: `CompetitionReferenceDto` gains `IsBuyerReady` + `ReadinessLevel`; `GetCompetitions` maps them from the catalog.
- `CompetitionCatalogReadinessTests.cs` (new): catalog readiness unit tests.

frontend (`dai`):
- `agent-run.model.ts`: `CompetitionReferenceDto` gains optional `isBuyerReady` + `readinessLevel`.
- `buyer-ready-competitions.ts`: filter now consumes `isBuyerReady` (frontend allowlist removed); fail-safe documented.
- `buyer-ready-competitions.spec.ts`: rewritten for metadata-driven behavior (test-first).

## Risks and gaps

- the backend `ReadinessLevel` strings and the frontend are stringly-typed; the catalog test pins the per-competition values, but there is no shared enum across the .NET/TS boundary (acceptable for one consumer; a generated contract is overkill now).
- landing coverage is still a static hardcoded list (aligned, with its own test) rather than driven from the competitions API. Driving landing/account copy from the same readiness source is a future packaging-from-metadata improvement, not done here to keep the change narrow.
- `/dev/artifacts` remains an unguarded internal route (uses routability); a dev-route guard is still recommended hardening (carried from the prior slice).

## Recommended next work

- drive landing coverage (and account plan language) from the competitions API `readinessLevel`/`isBuyerReady` so all buyer surfaces share one source of truth.
- a `/dev/artifacts` route guard.
- if readiness ever needs to be tenant/product-specific, move it from static catalog data to per-tenant config -- deferred until a tenant boundary exists.

## Recommended next slice

Outcome Reconciliation Runtime v1 (sequenced next) is unaffected by this change. The remaining packaging-from-metadata wiring (landing/account) and the `/dev/artifacts` guard are small follow-ups that can ride a later packaging slice.

## What was not changed

- no database schema or migration; no artifact contract; no buyer-projection semantics or buyer artifact meaning
- no new sport support; no source integration; no prompt/parser/confidence/posture/lean/signal/model/cost change
- no probe-refresh, outcome-reconciliation, pricing/Stripe/auth/tenant/dashboard/deployment change; no Jera change
- the FastAPI analyzer `_SUPPORTED_COMPETITIONS` (routability) is unchanged -- readiness is a separate, additive concept

## Verification

- .NET: targeted tests pass (16: SportsReference integration + the new catalog readiness tests); full solution builds.
- Angular: Vitest 47 pass (5 files), `ng build` exit 0.
- readiness future posture: code/config-backed today (catalog) is right for one product with no tenant boundary; a database-backed or per-tenant readiness store is deferred until tenants exist.
- `git diff --check` clean; added-line exact-path scan; non-ASCII scan 0 across changed/new code and vault docs.
