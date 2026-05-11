# sharp/public alternate provider discovery v1

**date:** 2026-05-11
**status:** discovery + decision only — no provider integration, no API keys, no production change.
**ladder rung under investigation:** source_substitution (rung 2) for the missing `sharp_public` primary signal.
**outcome:** **Sharp/Public Alternate Provider v1 is viable.** Primary recommendation: **SportsDataIO Odds API**. Runner-up: **VSiN (DraftKings-sourced HTML)** as a low-cost stop-gap if commercial sales cycle is gating.

---

## 1. summary

`sharp-public-provider-verification-results.md` ruled out exact_recovery from ActionNetwork. ActionNetwork's NBA playoff `odds[]` entries have null public/sharp percentage fields across all books at the unofficial endpoint. The ladder requires we exhaust source_substitution before reaching for adjacent or lateral proxies (Line Movement Proxy v1 remains deferred).

Two decision-grade facts came out of this discovery:

1. **DraftKings publishes NBA playoff splits in real time.** VSiN's data feed (sourced from DraftKings Sportsbook, 5-minute refresh) shows populated handle% and bets% for Monday May 11 and Tuesday May 12 playoff games. The data exists; the ActionNetwork failure is a single-provider gap, not a market-wide one.
2. **Two ownership groups are now off the table for source_substitution.** Sports Insights and BetIQ are sister products under the same parent as ActionNetwork (Better Collective, post-2021 acquisition). Picking either would not solve the root problem because they likely share the upstream data lake.

The cleanest commercial path is **SportsDataIO Odds API**, which aggregates 25+ books (DraftKings, FanDuel, BetMGM, Caesars, BetRivers, Circa, ESPN BET) and lists "Betting Matchups & Trends" plus betting splits as a covered product. It has a public free trial and a transparent commercial sales channel.

A practical stop-gap option is **VSiN's DraftKings-sourced HTML splits** (data.vsin.com/betting-splits/). It is single-book (DraftKings) and HTML-only, so it carries TOS friction for commercial product use. It is recommended only as a discovery/preview tier — not a v1 production source — pending VSiN TOS review.

---

## 2. why source_substitution is now the next rung

The Sharp/Public Fallback Ladder is the inherited contract. Exact recovery is empirically dead at ActionNetwork's free endpoint for the current playoff window (see verification doc section 6). The next rung is `source_substitution` — same signal, alternate source. We do not skip down to `fidelity_downgrade` or `adjacent_proxy` until source_substitution is also shown infeasible.

Fidelity downgrade — building a coarser "directional sharp sentiment" out of the still-available fields on ActionNetwork (line movement, num_bets count, total handle proxy) — is reserved as a rung-3 fallback that lands if every viable source_substitution candidate is too expensive, too restricted, or shares ActionNetwork's data lake.

Line Movement Proxy v1 stays deferred. `line_movement` is `adjacent_proxy / adjacent` on the ladder. It cannot equivalently substitute for `sharp_public` (it answers "did the price move," not "what was the public/sharp split").

---

## 3. provider comparison table

| provider | data type | book scope | NBA playoffs verified | api or html | terms / access | est. cost | confidence permission ceiling on the ladder | integration complexity |
|---|---|---|---|---|---|---|---|---|
| **SportsDataIO Odds API** | bets % and handle % per side, per market | 25+ books aggregated (DraftKings, FanDuel, BetMGM, Caesars, BetRivers, Circa, ESPN BET, …) | partial — provider product page claims splits coverage, NBA playoff specifics undisclosed without trial run | REST API, documented, OAuth-style key | commercial license, free trial available, sales contact required for splits-tier pricing | reported tier floor ~$400/mo for live sports data; splits-specific tier needs sales quote | `source_substitution / near` — `confidence_mostly_preserved`, `aggressive_allowed_if_corroborated` | **medium** — typed client, secret management, retry/cache, schema mapping |
| **VSiN (DraftKings-sourced)** | bets % and handle % per side, per market | single book (DraftKings Sportsbook) | **yes — verified live**: May 11 NBA playoff games show populated splits at data.vsin.com | HTML-only, no API, 5-min refresh | TOS not explicit on public page; commercial scraping likely requires VSiN/DraftKings license | scraping is "free" but legal cost is real; VSiN PRO subscription exists | `source_substitution / near` if licensed; `fidelity_downgrade / partial` if reduced to single-book directional read | **low (technical) / high (legal)** — scraping is easy, license clearance is the work |
| **Split Labs** | bets % and handle %, multi-book aggregated | multiple sportsbooks aggregated | not explicitly verified for NBA playoffs | HTML dashboard, no API stated | consumer subscription | $14.99–$24.99/mo (consumer tier); commercial license unclear | `source_substitution / near` if commercial license obtainable | **high** — no API; scraping a subscription product is a different TOS posture than scraping a free site |
| **Outlier (outlier.bet)** | public betting percentages | unclear (web product) | not verified | mobile-first product, web view exists | consumer-tier product | consumer subscription | `source_substitution / near` if API exists; otherwise n/a | **unknown** — no public API documentation found |
| **OpticOdds DraftKings API** | odds, alternate markets, player props | DraftKings (and many others) | n/a for splits — no explicit splits product on their page | REST API | commercial license, pricing on request | enterprise tier, no public price card | does not provide splits — **rejected** | n/a |
| **Unabated API** | odds, consensus "Unabated Line" (sharp-blended price) | 25+ books for odds; no splits | n/a — no splits product | REST + WebSocket API | commercial license | **$3,000/mo personal floor**, custom commercial | does not provide splits — **rejected** | n/a |
| **Pinnacle API** | odds, "public vs sharp betting patterns" hinted in marketing | Pinnacle only | n/a — see access | REST API | **closed to general public since 2025-07-23**; bespoke partnerships only | unknown, multi-month sales cycle | best in class signal *if* obtainable, but inaccessible — **rejected for v1** | n/a |
| **Sports Insights / BetIQ** | bets % and handle % | aggregated | likely affected by same upstream gap as ActionNetwork | API / dashboard | commercial license | tiered subscription | **rejected**: sister property of ActionNetwork under Better Collective. Sharing the upstream data lake defeats the substitution. | n/a |
| **Don Best / Bet The Number** | sportsbook odds feed (B2B) | many | n/a — not a splits product | enterprise API | enterprise license | enterprise pricing | does not provide public/sharp splits as a core product — **rejected** | n/a |
| **Sportradar / Genius Sports** | enterprise betting intelligence, integrity feeds | many | likely yes | enterprise API | enterprise contract, multi-month sales | **enterprise floor ~$50k+/yr** | best coverage but cost and sales cycle gate v1 — **deferred** | very high |
| **ActionNetwork (`fidelity_downgrade` from remaining fields)** | line movement, num_bets count, partial-period splits if any non-null | n/a | yes — fields are present, just no public/sharp percentages on full-game lines during playoffs | existing client | already integrated | $0 incremental | `fidelity_downgrade / partial` — `confidence_dampened`, `aggressive_blocked` | low — extend existing client to surface alternate fields as a degraded signal |

---

## 4. recommended provider

**SportsDataIO Odds API.**

Why this is the right pick:

- **It is a real commercial provider.** Public REST API, free trial, clear sales channel, and a TOS structured for product use. None of the legal-gray scraping issues that gate VSiN or Split Labs for v1.
- **It aggregates across the right book set.** DraftKings, FanDuel, BetMGM, Caesars, BetRivers, Circa, BetOnline, Fanatics, ESPN BET, "and more." Even if any one book is null on a given game, the aggregate likely has data. This is the architectural inverse of ActionNetwork's single-book pinning failure.
- **It is league-portable.** NFL, NBA, MLB, NHL covered now. The next two niches DAI plans to land (NFL, MLB) inherit the same client.
- **Trial path is concrete.** SportsDataIO offers a 30-day free trial. The verification step before signing a contract is a one-developer-day investigation: hit the splits endpoint for a current NBA playoff game and confirm populated bets% and handle%.

What this slice does NOT confirm:

- That the SportsDataIO `BettingSplits` endpoint *specifically* covers NBA playoff games with populated percentages. Their NBA coverage page and integration guide page were both light on this detail. **The Sharp/Public Alternate Provider v1 implementation slice must begin with a one-developer-day API verification against the free trial** before any client code is written. If verification fails, fall back to runner-up.

---

## 5. runner-up provider

**VSiN (DraftKings-sourced splits) — as a stop-gap, not a v1 production source.**

Why it is the runner-up:

- **Data is verified to exist for NBA playoffs.** Confirmed during this discovery against `data.vsin.com/nba/betting-splits/` — May 11 and 12 NBA playoff games show populated handle% and bets% on spread, total, and moneyline. The data is real and refreshes every 5 minutes.
- **Single-book scope is acceptable as a degraded signal.** DraftKings is one of the two largest US sportsbooks. A DraftKings-only sharp/public split is narrower than ActionNetwork's aggregate would be, but it is far better than null. On the ladder this is `source_substitution / near` if formally licensed; `fidelity_downgrade / partial` if used as a single-book directional read without explicit license.
- **It is the cheapest "yes" available.** No enterprise contract. The work is HTML parsing.

Why it is *not* the primary:

- **No API.** HTML parsing of a third-party media site introduces unbounded schema drift and a fragile production dependency.
- **TOS is unclear.** The VSiN splits page does not carry a visible commercial-use prohibition in the content sampled here, but VSiN's general site TOS and DraftKings's downstream licensing posture must be reviewed by legal before this becomes a v1 production source. Treating it as the primary without that clearance is the kind of shortcut the project's audit-before-redesign rule explicitly prevents.

VSiN may still be useful as a **diagnostic comparison source** during Sharp/Public Alternate Provider v1 implementation — running a SportsDataIO call against a parallel VSiN scrape on a small N of test games gives a sanity check that the API result tracks what the public splits page shows.

---

## 6. providers rejected and why

- **Sports Insights / BetIQ.** Same parent as ActionNetwork (Better Collective, since 2021). Likely shares the upstream feed that produced the null playoff fields. Picking a sister product is not a real source_substitution; it is the same source under a different brand.
- **OpticOdds.** Excellent DraftKings odds and player-props aggregator but no advertised public/money splits product. Wrong signal.
- **Unabated.** $3,000/mo personal floor and no public/money splits — their value is the "Unabated Line" (consensus sharp-leaning price). Adjacent at best on the ladder; expensive for a signal they do not actually provide.
- **Pinnacle API.** Closed to general public since 2025-07-23. Bespoke commercial partnerships only. Multi-month access cycle, not v1 viable.
- **Don Best / Bet The Number.** Sportsbook odds B2B provider. No public/money splits as a core product.
- **Sportradar / Genius Sports.** Best-in-class enterprise coverage. Enterprise contract floor and sales cycle gate v1. Hold for a later slice if SportsDataIO does not work out and the budget bar moves.

---

## 7. minimum viable data contract

The slice that implements source_substitution should produce a typed record. Sketched as a target contract (do not implement in this slice):

```text
SharpPublicSplitRecord {
  providerName:           string   // e.g. "sportsdata_io", "vsin_dk", "actionnetwork_an"
  providerSignalType:     string   // "aggregated_multi_book" | "single_book_dk" | "single_book_fd" | ...
  eventId:                string   // provider-native event identifier
  league:                 string   // "nba" | "nfl" | "mlb" | ...
  marketType:             string   // "spread" | "moneyline" | "total"
  side:                   string   // "home" | "away" | "over" | "under"
  publicBetPercentage:    int?     // bets % on this side (nullable: provider may not surface)
  publicMoneyPercentage:  int?     // handle % on this side (nullable: provider may not surface)
  sharpSide:              string?  // "home" | "away" | null — derived only when both percentages are populated and the divergence threshold is met
  sourceUpdatedUtc:       datetime // provider's own as-of timestamp
  retrievedUtc:           datetime // platform retrieval timestamp
  sourceUrl:              string?  // provenance for audit and TOS attribution
  sourceReference:        string?  // optional opaque provider reference (request id, internal event key)
  confidenceGrade:        string   // "strong" | "usable" | "directional_only_without_confirmation" | "unavailable"
  missingReason:          string?  // matches existing SharpPublicLookupResult vocabulary plus new values
}
```

Notes for the implementation slice:

- `providerName` is the load-bearing field. Without it, calibration cannot grade providers independently over time. **This is the artifact provenance requirement and must not be omitted.**
- `sharpSide` derivation is intentionally pushed to discern-phase code (`SignalQualityEvaluator`), not encoded by the provider client. The client surfaces the raw percentages; the platform decides what counts as "sharp divergence."
- `providerSignalType` is what lets the fallback ladder grade a single-book substitution (`fidelity_downgrade / partial`) differently from an aggregated multi-book substitution (`source_substitution / near`). The discern-phase ladder code should consume this field.
- The `missingReason` vocabulary should extend, not replace, the existing `SharpPublicLookupResult.MissingReason` set so existing telemetry continues to work for ActionNetwork while the new provider is added alongside.

The record is stored alongside the existing `SignalAvailabilityRecord` for `sharp_public`. The new client emits the record; downstream `SignalQualityEvaluator` and `SignalFollowUpEvaluator` consume it the same way they consume the current `SharpPublicContext`.

---

## 8. artifact provenance requirement

This is the architecture requirement the prompt called out explicitly. Every persisted sharp/public split must record:

1. **`providerName`** — which provider supplied this percentage for this run.
2. **`providerSignalType`** — single-book vs aggregated; this drives ladder classification.
3. **`sourceUpdatedUtc`** — the provider's as-of timestamp, not just our retrieval time.
4. **`sourceUrl` or `sourceReference`** — an audit anchor so any artifact can be traced back to the provider request that produced it.

These four fields land on the artifact regardless of whether the run grounded or missed the signal. On a miss, `providerName` captures which provider was attempted; future calibration can then ask questions like "is provider X failing more on NBA playoff games than provider Y?"

Without provenance, calibration cannot grade providers independently. It can only grade "the sharp_public signal" as a monolith — and that is what the existing platform already does, which is exactly what produced the current null-playoff gap with no diagnostic angle.

---

## 9. proposed implementation slice

**Sharp/Public Alternate Provider v1**

Phased so the cheapest verification step gates the larger client work.

### Phase A — paid trial verification (≤ 1 developer-day)

1. Sign up for SportsDataIO 30-day free trial.
2. Hit the NBA Betting Splits endpoint for one or more current playoff games.
3. Compare populated fields to the canonical record:
   - `publicBetPercentage` per side present?
   - `publicMoneyPercentage` per side present?
   - per-sportsbook breakdown available, or only aggregated?
   - `updated_at` timestamp present?
4. **Decision gate:**
   - If yes → proceed to Phase B with SportsDataIO as primary.
   - If no (NBA playoff data is null at SportsDataIO too) → re-scope to VSiN + legal review, or drop to Sharp/Public Fidelity Downgrade v1.

No production code is written in Phase A. The verification is shell scripts + curl + a short notes file under `04 Products/sports-v1/provider-investigations/`.

### Phase B — minimal client + artifact provenance

1. New `SportsDataIoSharpPublicClient` in `dai/platform/dotnet/DevCore.Api/Sports/` modeled on the existing `ActionNetworkClient`.
2. New `SharpPublicSplitRecord` typed contract (see section 7) added to `AgentRunContracts.cs`.
3. New typed `HttpClient` registration in `Program.cs` with secret management for the API key.
4. `SportsRetriever.RetrieveAsync` learns a two-provider lookup pattern: try ActionNetwork first (current path), fall back to SportsDataIO on missing-with-reason. Both attempts are recorded as `SharpPublicSplitRecord` entries with explicit `providerName`. The artifact gains a `SharpPublicProvenance` array showing what was attempted and what landed.
5. `SignalQualityEvaluator` learns to consume `providerSignalType` so a single-book result reads as `fidelity_downgrade / partial` even when it grounds successfully.
6. Tests:
   - Provider A grounds → record carries `providerName="actionnetwork"`, ladder unchanged.
   - Provider A misses, Provider B grounds → record carries `providerName="sportsdata_io"`, ladder classifies as `source_substitution / near`.
   - Both providers miss → existing `unavailable_with_reason` path preserved, but provenance shows both attempted.

### Phase C — calibration integration (later slice, not v1)

`SportsEvaluator` learns to consume `ConfidencePermission` and `PosturePermission` from the ladder classification rather than only `confidence_effect`. This is the separate "Calibration Ladder Wiring v1" slice that was previously documented as deferred.

---

## 10. risks and open questions

- **Phase A may invalidate the primary recommendation.** It is plausible that SportsDataIO's NBA splits coverage is null on the same playoff games that ActionNetwork is null on (same upstream books, same root cause). The verification step exists precisely because the public marketing pages do not disclose playoff-specific availability. If verification fails, the slice changes from "wire SportsDataIO" to "downshift to fidelity_downgrade or legal-clear VSiN."
- **Better Collective ownership concentration.** Action Network, Sports Insights, and BetIQ all sit under the same parent. Any future M&A in the splits-aggregator space could collapse other independent-looking providers onto the same data lake. The artifact provenance requirement (section 8) is the protection — calibration can detect this drift over time even if procurement misses it.
- **Single-book signal quality.** A DraftKings-only substitution is structurally narrower than ActionNetwork's full aggregate. Sharp action visible at DraftKings is not the same as sharp action visible across the full book set. The ladder's `providerSignalType` field captures this; `SignalQualityEvaluator` must use it to keep `single_book_dk` results at `fidelity_downgrade / partial` rather than `source_substitution / near`.
- **Provenance retrofit.** Existing artifacts persisted under `OutputJson` predate `SharpPublicSplitRecord`. Backfilling provenance retroactively is impossible. Calibration queries that grade providers must include "or null = pre-provenance" as a known cohort.
- **TOS friction on the runner-up.** VSiN's commercial-use posture is not explicit on the splits page. The runner-up cannot ship to production without a legal pass, even though the data is publicly visible and refreshes on a public URL.
- **Cost compounds at the niche level.** SportsDataIO covers NFL/NBA/MLB. The minimum subscription tier likely covers all three at one price, but the splits add-on may be a separate line item. The Phase A verification should also confirm whether splits are bundled with the base Odds API or tier-gated.
- **Provider failure-mode taxonomy must extend.** The existing `SharpPublicLookupResult.MissingReason` vocabulary was designed against ActionNetwork's failure modes. SportsDataIO will have its own (rate limit, key expired, plan not entitled, sport not covered on plan). The implementation slice extends the enum rather than retrofitting ActionNetwork's reasons onto a different provider.
- **Phase B touches retrieval ordering.** The current `SportsRetriever.RetrieveAsync` is synchronous and single-source per signal. Adding a two-provider fallback changes the run timing and exception-handling surface. The implementation slice should keep total retrieval latency under the existing P95 budget — a parallel-attempt strategy or a strict "fallback only on miss" sequential strategy are both viable; pick one explicitly.

---

## decision

Per the decision rule in the prompt:

> If one provider clearly supports NBA playoffs plus future NFL/NBA/MLB coverage with usable terms and reasonable integration cost, recommend Sharp/Public Alternate Provider v1.

A viable provider exists with the right scope, terms structure, and integration cost: **SportsDataIO Odds API**. A real verification gate is required (Phase A) before committing the slice budget. Recommend **Sharp/Public Alternate Provider v1**, executed in three phases with the trial verification first.

`Sharp/Public Fidelity Downgrade v1` becomes the named fallback if Phase A invalidates the primary recommendation.

`Line Movement Proxy v1` remains deferred — it is `adjacent_proxy / adjacent` on the ladder and cannot equivalently substitute for `sharp_public`.

---

## references

- prior investigation: `sharp-public-provider-investigation-v1.md`
- prior verification: `sharp-public-provider-verification-results.md`
- ladder doctrine: `02 Platform/architecture/cognitive-factory/signal-fallback-ladder.md`
- existing client: `dai/platform/dotnet/DevCore.Api/Sports/ActionNetworkClient.cs`
- existing contract: `dai/platform/dotnet/DevCore.Api/AgentRuns/AgentRunContracts.cs` (`SharpPublicLookupResult`, `SignalAvailabilityRecord`, `SignalFollowUpRecord`)
- skill: `dai/.claude/skills/dai-signal-follow-up-diagnostics/`

external sources reviewed for this discovery:

- SportsDataIO product pages: sportsdata.io (live odds API, NBA coverage, betting integration guide)
- VSiN splits: data.vsin.com/betting-splits/, data.vsin.com/nba/betting-splits/ (verified NBA playoff data live during this discovery)
- DraftKings: dknetwork.draftkings.com/draftkings-sportsbook-betting-splits/
- Unabated: unabated.com/get-unabated-api
- OpticOdds: opticodds.com/sportsbooks/draftkings-api
- Pinnacle API status: pinnacleapi.github.io / Pinnacle developer docs (confirmed closed to general public since 2025-07-23)
- Split Labs: splitlabs.net
- Better Collective acquisition history: globenewswire and sportshandle press records (confirms Action Network is a Better Collective property)
