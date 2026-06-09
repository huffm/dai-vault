# Pre-Support Buyer Validation Matrix v1

**date:** 2026-06-09
**status:** support-readiness classification. docs only. no runtime code, no source integration, no prompt/parser/schema/contract/buyer-projection/buyer-copy/confidence/posture/lean/model/cost change, no public support-claim change.
**scope:** classify each target sport/league by buyer-support readiness before DAI claims or expands support for it.

## Purpose

The factory can produce analyzer text for several competitions, but "can generate text" is not "buyer support." Support means the factory can produce buyer-credible, grounded, safe, repeatable artifacts for a sport at acceptable cost with known limitations. This slice classifies NBA, MLB, NFL, CFB, NCAAB men, NCAAB women, WNBA, and NHL by readiness so product claims do not outrun validation.

Primary question: which sports are ready for buyer-facing artifacts now, which are smoke-only, which need source/model/protocol work, and which should stay deferred until demand or data quality justifies support?

## Scope

- read-only inspection of supported competitions, the competition catalog, source clients, analyzer paths, seeded reference data, and existing validation/calibration artifacts.
- produce a per-sport readiness matrix and product-support-claims guidance.

## Non-goals

- no source integration, prompt, parser, schema, artifact-contract, buyer-projection, or buyer-copy change.
- no confidence/posture/lean/signal/model/cost change; no probe-refresh; no outcome-reconciliation implementation; no outcome-provider choice.
- no public support claims changed in the frontend; no pricing/Stripe/auth/tenant/dashboard/deployment change. no Jera change. no push.

## Input documents / code inspected

- `services/agent-service/app/routes/sports.py` (`_SUPPORTED_COMPETITIONS`, analyzer dispatch).
- `platform/dotnet/DevCore.Api/Sports/CompetitionCatalog.cs` (the authoritative competition + expected-signal map).
- source clients: `OddsScheduleClient`, `OddsMarketClient` (the-odds-api), `MlbStarterClient` (statsapi.mlb.com), `EspnBasketballScheduleClient` (ESPN), `ActionNetworkClient` (sharp/public).
- `DevCore.Data/SeedData/SportsSeedData.cs` (+ college seed data) for seeded leagues/teams.
- existing vault evidence: Fresh Buyer Artifact Validation v1/v2, Evidence-Sufficiency Band Gate v1, Confidence Calibration Rules v1, Artifact Cost Guardrails v1, Outcome Reconciliation Contract v1, and the calibration artifacts under `calibration/`.

## Support-readiness classification definitions

1. Buyer-ready validated -- fresh full-chain artifacts exist, buyer projection works, safety clean, evidence-sufficiency behavior validated, cost known, source limits understood.
2. Buyer-ready limited -- credible artifacts possible, but coverage is narrow/seasonal/source-limited; support only with explicit constraints.
3. Smoke-level only -- a generation path exists/can be tested, but not enough validated artifacts or source guarantees to claim buyer support.
4. Contract-ready, source-limited -- doctrine + artifact contract support the sport, but source retrieval/identity/schedule/signal availability is insufficient.
5. Deferred -- do not support until demand, source coverage, or season availability justifies the work.
6. Not supported -- no credible path exists in code/source today without meaningful new implementation.

## Current competition / source map

Routable competitions in code (FastAPI `_SUPPORTED_COMPETITIONS` and `CompetitionCatalog`), with expected grounded signals (= `MaxGroundedSignals`) and odds-api key:

- nfl (NFL, football/pro): signals [market, sharp_public]; odds key americanfootball_nfl.
- ncaaf (CFB, football/college): signals [market, sharp_public]; odds key americanfootball_ncaaf.
- nba (NBA, basketball/pro): signals [rest_schedule, market, sharp_public]; odds key basketball_nba.
- ncaamb (NCAAB men, basketball/college): signals [rest_schedule, market, sharp_public]; odds key basketball_ncaab (ActionNetwork maps ncaamb -> "ncaab" -- provider naming differs).
- mlb (MLB, baseball/pro): signals [starting_pitching]; odds key baseball_mlb.
- "College Baseball": catalog entry with `Code = null`, "not available yet" -- non-routable placeholder.

Analyzer paths (3): `analyze_football` (nfl, ncaaf), `analyze_basketball` (nba, ncaamb), `analyze_mlb` (mlb).

Source-signal coverage: market + schedule via the-odds-api (all five); sharp_public via ActionNetwork (all five, often missing in practice); rest_schedule via ESPN (basketball only: nba, ncaamb); starting_pitching via MLB statsapi (mlb only).

Seeded reference data (Sports/Teams): nfl, nba, mlb, ncaaf, ncaamb only.

Fresh full-chain validated artifacts: NBA (21 artifact files, 15 calibration reports) and MLB (23 artifact files, 4 reports) only. NFL, NCAAF, NCAAMB have zero validated artifacts.

Absent entirely from code/catalog/seed/analyzer: WNBA, NHL, NCAAB women.

Internal-name mapping: NBA->nba, MLB->mlb, NFL->nfl, CFB->ncaaf, NCAAB men->ncaamb, NCAAB women->(none), WNBA->(none), NHL->(none).

## Per-sport validation matrix

Columns: code path | full-chain proven | buyer projection proven | source coverage | typical evidence sufficiency | identity reliability | season now (2026-06-09) | classification | product posture.

- NBA: yes | yes | yes | rest(ESPN)+market(odds)+sharp(AN, usually missing) | moderate (evidenceRichness ~2) | pro, reliable | Finals then off-season (narrow slate, 1 game/14d) | Buyer-ready validated | support with caveat (seasonal).
- MLB: yes | yes | yes | starting_pitching(MLB statsapi); market not an expected grounded signal | thin (evidenceRichness 1; band-gated to advertised Medium) | pro, reliable | in season (full slate) | Buyer-ready validated | support with caveat (single-signal thinness, honestly gated).
- NFL: yes | no | no | market(odds)+sharp(AN); no sport-specific grounding signal | likely thin (often market-only -> 1, or 0 if sharp missing) | pro, reliable | off-season | Smoke-level only | internal smoke only; defer buyer claim until in-season validation.
- CFB (ncaaf): yes | no | no | market(odds)+sharp(AN) | likely thin | college, ambiguous names | off-season | Smoke-level only | internal smoke only; defer; needs identity caution.
- NCAAB men (ncaamb): yes | no | no | rest(ESPN)+market(odds)+sharp(AN) | moderate-to-thin | college, ambiguous names; provider naming differs (ncaab) | off-season | Smoke-level only | internal smoke only; defer until winter validation.
- NCAAB women: no code path | no | no | none (no analyzer code/source/seed) | n/a | worst (naming ambiguity + thin provider coverage) | off-season | Deferred (Not-supported in code today) | defer until demand + source/identity groundwork.
- WNBA: no code path | no | no | none | n/a | pro, but unbuilt | in season (but no code) | Deferred (Not-supported in code today) | defer until demand justifies the build.
- NHL: no code path | no | no | none | n/a | pro, but unbuilt | roughly off-season | Deferred (Not-supported in code today) | defer until demand appears.

Cost telemetry is available for every routable competition (one gpt-4o-mini call per run, ~$0.0006-0.0007 per artifact, log-only -- Artifact Cost Guardrails v1). Stable game/event identity is absent for all sports (Outcome Reconciliation Contract v1) and is most risky for college/women's sports.

## Sport-by-sport findings

- NBA: the most validated basketball case. Full-chain proven across multiple slices; buyer projection clean; 0 unsafe/tout hits; evidence-sufficiency band gate validated (moderate evidence, not capped). Caveat: `sharp_public` is consistently missing, so the practical richness is ~2 of 3, and the slate is seasonal (Finals now, then off-season). Posture: support with caveat.
- MLB: the most validated thin-evidence case. In season with a full daily slate; full-chain proven; 0 unsafe hits; structurally single-signal (`starting_pitching` only, maxGrounded 1), so raw High is correctly capped to advertised Medium by the band gate. Repetition is controlled (validated). Posture: support with caveat (be explicit that MLB reads are single-signal and presented at measured strength).
- NFL: code path, analyzer, source clients, and seeded teams exist, but no fresh validation and it is off-season, so no slate to validate against now. Signal set is only market + sharp_public with no sport-specific grounding signal, so reads may often be thin or market-only. Smoke-level only; validate a first buyer batch when the season returns.
- CFB (ncaaf): same football analyzer and signal set as NFL, plus college team-name ambiguity (the Outcome Contract flagged identity risk; the repo's own Oakland->Sacramento rename shows names drift even in pro leagues). Off-season, unvalidated. Smoke-level only with identity caution.
- NCAAB men (ncaamb): same basketball analyzer/signal set as NBA (rest+market+sharp), plus college identity ambiguity and a provider naming difference (ActionNetwork "ncaab"). Off-season, unvalidated. Smoke-level only; validate in winter.
- NCAAB women: no analyzer routing, no source coverage, no seeded teams. The basketball analyzer could in principle extend to it, but women's college has the worst naming ambiguity and thinnest provider coverage, so source + stable-identity groundwork must precede any work. Deferred (effectively not supported in code today).
- WNBA: in season, but entirely unbuilt (no competition code, analyzer routing, source mapping, or seed data). Plausible future sport; deferred until demand justifies the build.
- NHL: entirely unbuilt and lower priority; roughly off-season. Deferred until demand appears.

## Support claims guidance

- soft-launch / private validation: MLB (in season, validated) is the strongest "support with caveat" today; NBA (validated) can be included with an explicit seasonal caveat. Present both as measured decision support, never as strong picks.
- hidden or "coming later": NFL, CFB, NCAAB men -- code exists but is unvalidated and off-season; do not claim buyer support; surface only after an in-season validation batch.
- internal only: NFL/NCAAF/NCAAMB direct-analyze smokes; do not expose to buyers.
- manual artifact review required before any buyer-facing output: the first NFL, NCAAF, and NCAAMB buyer batches must be manually reviewed (safety, repetition, evidence sufficiency) when their seasons return.
- avoid overclaiming: never list WNBA, NHL, or NCAAB women as supported (no code path); never imply "five sports supported" -- only two (NBA, MLB) are buyer-validated. The other three routable competitions are smoke-only.

## Risks and gaps

- validation gap: 3 of 5 routable competitions (NFL, CFB, NCAAB men) have never been buyer-validated; their season is not active now, so validation is time-gated.
- thin-evidence dominance: MLB (1 signal) and likely NFL/CFB (market-only when sharp is missing) lean structurally thin; the band gate keeps this honest but limits how strong reads can be advertised.
- signal flakiness: `sharp_public` (ActionNetwork, unofficial endpoint) is frequently missing across sports, reducing evidence richness everywhere.
- identity risk: no stable external game/event id on runs yet (Outcome Reconciliation Contract v1); college and women's naming ambiguity makes display-name matching unsafe for those sports.
- overclaim risk: a code path or a schedule source can be mistaken for buyer support; this matrix exists to prevent that.

## Recommended next work

- per-sport in-season validation batches: a Football Pre-Support Validation (NFL + CFB) in the fall and an NCAAB validation in winter, each gated by manual review.
- stable game/event id capture (from the Outcome Reconciliation Contract) before any automated reconciliation, prioritized for college/women's sports if ever enabled.
- per-sport signal catalog / source-coverage review (especially `sharp_public` reliability and whether football needs a sport-specific grounding signal beyond market).
- cost telemetry sink (ledger entry 22) so per-sport unit economics become queryable.
- buyer packaging by sport: the frontend should expose only readiness-classified sports; do not list smoke-only or deferred sports as supported.

## Recommended next slice

Outcome Reconciliation Runtime v1 (planned step 7) remains the sequenced next implementation. In parallel, when football season returns, a Football Pre-Support Validation v1 should move NFL/CFB from smoke-only toward buyer-ready using the existing pipeline (no new doctrine needed). Neither requires new sources beyond what the catalog already maps.

## What was not changed

- no runtime code, source integration, prompt, parser, schema, artifact contract, or buyer projection
- no buyer copy or public support claims (frontend untouched)
- no confidence/posture/lean/signal/model/cost change; no probe-refresh; no outcome-reconciliation implementation; no provider choice
- no Postgres/migration/pricing/Stripe/auth/tenant/dashboard/deployment change
- no Jera change

## Verification / unknowns

- all code claims are grounded in `_SUPPORTED_COMPETITIONS`, `CompetitionCatalog`, the source clients, the seed data, and the validation/calibration artifacts (read this slice).
- not re-run this slice: large fresh generation batches (deliberately, to limit model spend); NBA/MLB classifications rely on the prior fresh validations (v1/v2 + band gate), not new generation.
- football/college signal-richness expectations (thin/market-only) are projections from the catalog signal sets and `sharp_public` history, not from validated in-season artifacts; confirm with an in-season validation batch.
