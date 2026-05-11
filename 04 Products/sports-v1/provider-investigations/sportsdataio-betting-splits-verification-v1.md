# sportsdataio betting splits verification v1

**date:** 2026-05-11
**status:** documentation-grade verification only — no API key available locally; trial data is provider-scrambled by design; runtime data availability for NBA playoffs is NOT yet verified.
**ladder rung under test:** source_substitution (rung 2) — does SportsDataIO supply the same-signal `BetPercentage` + `MoneyPercentage` data that `sharp_public` requires?
**outcome:** **Conditional proceed.** SportsDataIO's documented `BettingSplit` schema qualifies as `source_substitution / near`. Real-runtime playoff data availability has not been verified — that step requires an unscrambled (paid or sales-issued) API key. Recommend proceeding to **Sharp/Public Alternate Provider v1 Phase A** (paid-key verification) before any client code is written; do not commit to the slice budget on documentation alone.

---

## 1. summary

The prior discovery doc named SportsDataIO Odds API as the primary source_substitution candidate, conditional on a Phase A verification that the NBA Betting Splits endpoint returns populated playoff data. This slice does the pre-Phase-A work that does not require an API key:

- locate the actual endpoint family and URL pattern
- confirm authentication mechanism
- confirm the response schema (which fields are returned)
- understand the trial-vs-paid coverage model
- identify any provider-side caveats that affect playoff-specific availability

What this slice can confirm: the documented schema is correct, the URL pattern is known, and `BetPercentage` + `MoneyPercentage` fields exist as named objects. The signal type matches `sharp_public`.

What this slice cannot confirm without a real key: whether NBA playoff games on current dates actually return populated `BetPercentage` and `MoneyPercentage` values, or whether they too come back null the way ActionNetwork's bookId=15 response did. SportsDataIO publishes a critical caveat that drives this risk: "*NBA Betting Splits begin to populate shortly after Game lines are posted and action is taken by the sportsbooks. Betting Splits are available at the discretion of sportsbooks making the data available.*" Same root dependency as ActionNetwork — multiple sportsbooks instead of one, but the same upstream sources.

Per the verification protocol, I did not sign up for a trial or generate keys. The free trial explicitly states "data in the free trial is scrambled for demonstration purposes," so a free trial would only validate the schema (already done) and not the playoff availability question. Real verification requires a paid plan or an unscrambled trial issued by the sales team.

---

## 2. endpoint investigated

**Base URL:** `https://api.sportsdata.io/v3/nba/odds/json/`

**Endpoint family for betting markets:** `BettingMarkets*` — most notably:

- `BettingMarketsByGameID/{gameId}`
- `BettingMarketsByEvent/{eventId}`
- `BettingMarketsByScoreID/{scoreId}` (documented for NFL; pattern applies to NBA per SportsDataIO architecture-is-uniform-across-sports stance)
- `BettingMarketsByMarketType/{type}`
- `BettingPlayerPropsByGameID/{gameId}`

The `BettingSplit` object is a child of `BettingMarket` (joined by `BettingMarketID`). The most likely access path is one of:

1. `BettingMarketsByGameID/{gameId}?include=available` — `BettingSplit` records may be returned inline as a nested array on each market or outcome.
2. A dedicated `BettingSplitsByGameID/{gameId}` or `BettingSplitsByMarketID/{marketId}` endpoint that returns the splits as a flat list.

The exact endpoint name for the splits payload is not in any public-facing page I could load. The Phase A verification step (with a key) must confirm which of the two patterns SportsDataIO uses. Both patterns return the same fields.

**Parameter notes:**

- `include=available` is the recommended parameter for speed; `include=unlisted` is the slower, more-detailed alternative. Phase A must test both because `available` may strip the splits subtree.

**Authentication:**

Two equivalent forms (provider-documented):

- Header: `Ocp-Apim-Subscription-Key: {key}`
- Query param: `?key={key}`

Either is acceptable. The header form is preferred for production code so the key does not appear in proxy logs or HTTP referer headers.

**Versioning:** `v3` is the current published version family.

---

## 3. api access and tier findings

**Free trial:** Available without sales contact, but data is "scrambled for demonstration purposes." This blocks the playoff-availability verification entirely — even if every endpoint responds, the values are not real. The schema is real; the percentages are not.

**Production / paid plan:** Required for unscrambled data. Pricing is not published on the public site. Other sources during the prior discovery indicated commercial tier floor in the "~$400/month for live sports data" range, but splits-specific tier ("Trends & Insights" / "Trends, Matchups & Insights") may be priced separately. Sales contact required for an authoritative quote.

**Replay:** SportsDataIO offers a "Replay" mode that lets a developer replay historical sports data through a sandbox version of the live API. Replay against a known regular-season NBA date with populated splits would let us verify the schema with real values **without** a paid plan — but it would still not verify the current 2026 NBA playoff blackout question, only the schema integrity.

**Tier disambiguation (open question for Phase A):**

The SportsDataIO product portfolio mentions four betting-data product names. The slice must confirm which one carries splits:

- "Live Odds Feeds" — odds only; almost certainly does **not** include splits.
- "Pricing, Trading & Settlement" — settlement-focused; unlikely to include splits.
- "Matchups, Trends & Insights" — likely the product that carries `BettingSplit`. The workflow guide phrase "Matchups, Trends & Splits" appears as a team-level analytics product though, which is a different thing from bet-distribution splits. Phase A must disambiguate.
- "Betting Data Integration Guide" mentions both "Betting Markets" and analytics products but doesn't bind splits to a specific tier in the public-facing copy.

**Result: tier-pricing for splits is undisclosed without sales contact.** Treat this as a hard Phase A item: ask sales explicitly which tier includes `BettingSplit` records, and confirm the tier price before any slice budget is approved.

---

## 4. verification commands or manual steps

Since no API key is available in local config, this slice produces a verification protocol for the developer who acquires a key. Run these after Phase A signup.

```bash
# 0. Set the key locally, never commit it.
export SPORTSDATAIO_KEY='<your-trial-or-paid-key>'

# 1. Find a current NBA game id from the schedule endpoint.
curl -sS -H "Ocp-Apim-Subscription-Key: $SPORTSDATAIO_KEY" \
  "https://api.sportsdata.io/v3/nba/scores/json/GamesByDate/2026-05-11" \
  | python -m json.tool | head -40

# 2. Try the inline-splits pattern first.
curl -sS -H "Ocp-Apim-Subscription-Key: $SPORTSDATAIO_KEY" \
  "https://api.sportsdata.io/v3/nba/odds/json/BettingMarketsByGameID/<GAME_ID>?include=available" \
  | python -m json.tool > out_available.json

curl -sS -H "Ocp-Apim-Subscription-Key: $SPORTSDATAIO_KEY" \
  "https://api.sportsdata.io/v3/nba/odds/json/BettingMarketsByGameID/<GAME_ID>?include=unlisted" \
  | python -m json.tool > out_unlisted.json

# 3. Search both responses for the splits fields. They are populated only if
#    BetPercentage and MoneyPercentage appear with non-null numeric values.
python -c "
import json
for path in ('out_available.json', 'out_unlisted.json'):
    data = json.load(open(path))
    found = []
    def walk(node, trail=''):
        if isinstance(node, dict):
            if 'BetPercentage' in node or 'MoneyPercentage' in node:
                found.append((trail, node.get('BetPercentage'), node.get('MoneyPercentage'),
                              node.get('BettingOutcomeType'), node.get('LastSeen')))
            for k, v in node.items():
                walk(v, trail + '/' + k)
        elif isinstance(node, list):
            for i, v in enumerate(node):
                walk(v, trail + f'[{i}]')
    walk(data)
    print(path, '— BettingSplit-bearing nodes:', len(found))
    for f in found[:10]: print(' ', f)
"

# 4. If splits do not appear inline, try a dedicated splits endpoint by guessing the path.
#    (Provider has not published this endpoint name in public docs.)
curl -sS -H "Ocp-Apim-Subscription-Key: $SPORTSDATAIO_KEY" \
  "https://api.sportsdata.io/v3/nba/odds/json/BettingSplitsByGameID/<GAME_ID>" \
  | head -c 400
```

Decision rule from step 3:

- ✅ **`BetPercentage` and `MoneyPercentage` appear with non-null numeric values for a current NBA playoff game** → SportsDataIO qualifies as `source_substitution / near`. Proceed to Sharp/Public Alternate Provider v1 Phase B.
- ⚠️ **`BetPercentage` / `MoneyPercentage` appear in the schema but are null for current playoff games** → same root failure as ActionNetwork. Reject SportsDataIO, escalate to runner-up (VSiN) or rung-3 fidelity_downgrade.
- ❌ **The fields are not returned at all on this tier** → the splits product requires a different tier; either upgrade or reject for v1.

If the developer has only a free trial key (scrambled data), they can still run the schema-shape check from step 3 — but they cannot answer the runtime-availability question. That requires unscrambled access.

**Important:** never commit the key. The verification artifacts (`out_available.json`, `out_unlisted.json`) should also stay out of git unless they are first manually sanitized of any identifying account markers.

---

## 5. games or dates checked

None at runtime in this slice. No API key was acquired; the verification commands above were not executed.

Documentation-only inspection:

- SportsDataIO NBA API documentation page (`sportsdata.io/developers/api-documentation/nba`)
- SportsDataIO NBA Workflow Guide (`sportsdata.io/developers/workflow-guide/nba`)
- SportsDataIO NBA Data Dictionary (`sportsdata.io/developers/data-dictionary/nba`)
- SportsDataIO Live Odds API marketing page (`sportsdata.io/live-odds-api`)
- SportsDataIO Free Trial page (`sportsdata.io/free-trial`)
- SportsDataIO API Resources page (`sportsdata.io/developers/apis`)
- SportsDataIO Betting Data Integration Guide (support and main-site copies)

Cross-reference from the prior discovery slice: ActionNetwork's `bookId=15` query on the documented Spurs-at-Timberwolves matchup (gameId 290435, 2026-05-10) returned `odds: []`. This matchup is the natural Phase A test target — if SportsDataIO returns populated splits for the same matchup, the substitution is validated against the exact gap that prompted the slice.

---

## 6. fields returned (from data dictionary)

This is what the documented schema looks like. None of these values are observed at runtime in this slice.

**`BettingSplit` object — same-signal payload:**

| field | definition | substitution role for sharp_public |
|---|---|---|
| `BetPercentage` | "Percentage of bets placed on a specific outcome" | direct equivalent of `spread_home_public` / `spread_away_public` (tickets) |
| `MoneyPercentage` | "Percent of Money on this Outcome" | direct equivalent of `spread_home_money` / `spread_away_money` (handle) |
| `BettingOutcomeTypeID` | int identifier | side index |
| `BettingOutcomeType` | name (`Home` / `Away` / `Over` / `Under`) | direct equivalent of our `side` |
| `Created` | timestamp first recorded | provenance — useful for staleness checks |
| `LastSeen` | most recent update timestamp | direct equivalent of `updatedAt` on ActionNetwork response |
| `BettingMarketID` | join key to parent `BettingMarket` | enables grouping all sides of a market |

**`BettingMarket` object — market metadata:**

| field | definition |
|---|---|
| `BettingMarketTypeID` | int |
| `BettingMarketType` | name (`Player Prop`, `Team Prop`, `Game Prop`, etc.) |
| `BettingBetType` | enum: `Moneyline`, `Spread`, `Total Points`, etc. — direct match for our `marketType` |
| `BettingPeriodType` | period scope (`Full Game`, `1st Half`, etc.) |
| `Updated` | last modification timestamp |

**`BettingOutcome` object — odds with sportsbook attribution:**

| field | definition |
|---|---|
| `BettingOutcomeType` | side name |
| `SportsBook` | book identifier — enables per-book attribution |
| `PayoutAmerican` / `PayoutDecimal` | odds |
| `Value` | line value (spread amount, total amount) |
| `Updated` | timestamp |

**Conclusion on field coverage:** the SportsDataIO `BettingSplit` schema carries `BetPercentage`, `MoneyPercentage`, `BettingOutcomeType` (side), the parent market type via `BettingMarket.BettingBetType`, and freshness timestamps (`Created` / `LastSeen`). This is the same-signal field set required for `source_substitution / near` per `signal-fallback-ladder.md`. Provenance can be reconstructed via `SportsBook` on the related `BettingOutcome`.

---

## 7. whether playoff split data is populated

**Not verified at runtime in this slice.** Verification requires an unscrambled API key, which is not available locally.

Three relevant pieces of evidence frame the risk:

1. **The data dictionary documents the fields.** The schema is real.
2. **SportsDataIO publishes the same dependency caveat that doomed ActionNetwork:** "Betting Splits are available at the discretion of sportsbooks making the data available." If sportsbooks aren't publishing during the NBA playoff window for any reason (regulatory, business, integrity), SportsDataIO inherits the gap.
3. **SportsDataIO's book set is wider.** ActionNetwork was pinned to a single book (FanDuel via `bookId=15`). SportsDataIO aggregates across 25+ books per their marketing. Even if a few books are dark, others may still publish. Probabilistically this is a meaningful improvement, but it is not a guarantee.

The honest answer is: **the schema qualifies, the runtime availability is a known open question.** Phase A of Sharp/Public Alternate Provider v1 must resolve it before any client code is written.

---

## 8. whether this qualifies as source_substitution / near

**Conditionally yes.**

By documented schema: the substitution is same-signal. `BetPercentage` and `MoneyPercentage` are direct equivalents of the existing ActionNetwork fields. Market type, side, and freshness timestamps are all present. Provenance via `SportsBook` is available.

By runtime data: not yet verified. If Phase A confirms populated playoff data, the classification stands at `source_substitution / near` with `confidence_mostly_preserved` and `aggressive_allowed_if_corroborated`. If Phase A reveals null playoff data, the classification drops — and the substitution fails entirely. There is no middle outcome at this rung; either the data is there or it isn't.

The discovery doc's `providerSignalType` field becomes important here: a single-book SportsDataIO response (only one of the 25+ books publishing) would be `single_book_*` and should be graded `fidelity_downgrade / partial`, not full `source_substitution`. The full `source_substitution / near` grade requires aggregated-multi-book grounding. This rule lives in `SignalQualityEvaluator` once the slice ships; it is not a SportsDataIO-side concern.

---

## 9. minimum viable production contract

The `SharpPublicSplitRecord` shape sketched in the discovery doc holds, with one refinement: the `providerSignalType` field's values must be enumerable from the SportsDataIO response.

| target field | sourced from | notes |
|---|---|---|
| `providerName` | constant `"sportsdata_io"` | provenance — required, never null |
| `providerSignalType` | derived: `aggregated_multi_book` if `len(distinct SportsBook) >= 3`, else `single_book_<sportsbook>` | drives ladder rung |
| `eventId` | provider `GameID` | join key |
| `league` | constant `"nba"` (per call) | |
| `marketType` | `BettingMarket.BettingBetType` mapped to `"spread"` / `"moneyline"` / `"total"` | mapper |
| `side` | `BettingSplit.BettingOutcomeType` mapped to `"home"` / `"away"` / `"over"` / `"under"` | mapper |
| `publicBetPercentage` | `BettingSplit.BetPercentage` | nullable |
| `publicMoneyPercentage` | `BettingSplit.MoneyPercentage` | nullable |
| `sharpSide` | derived in discern; not from provider | platform owns |
| `sourceUpdatedUtc` | `BettingSplit.LastSeen` | provider timestamp |
| `retrievedUtc` | platform-set | retrieval anchor |
| `sourceUrl` | request URL captured at retrieval (with key redacted) | audit |
| `sourceReference` | `BettingMarketID` | provider join key |
| `confidenceGrade` | derived by `SignalQualityEvaluator` from `providerSignalType` + population | platform owns |
| `missingReason` | extends existing vocab with provider-specific values (e.g. `sportsdataio_splits_null`, `sportsdataio_tier_not_entitled`, `sportsdataio_rate_limited`, `sportsdataio_key_invalid`) | additive |

The mapping itself is straightforward; the `providerSignalType` derivation is the load-bearing piece. Phase A verification must confirm `SportsBook` distinct-count is meaningful in real responses.

---

## 10. recommendation

**Conditional proceed to Sharp/Public Alternate Provider v1 Phase A.**

Phase A is the paid-key or unscrambled-trial verification step. It is in scope of the next slice, not this one. Execute it before Phase B (client implementation).

If Phase A confirms populated playoff splits → continue with Phase B per the discovery doc.

If Phase A reveals null playoff splits → reject SportsDataIO, escalate to:
- **Runner-up:** VSiN (DraftKings-sourced) under explicit legal review, or
- **Rung 3:** Sharp/Public Fidelity Downgrade v1 (extend ActionNetwork client to surface a degraded signal from still-available fields)

**`Line Movement Proxy v1` remains deferred.** It is `adjacent_proxy / adjacent` on the ladder regardless of what Phase A finds. It does not enter the recommendation set here.

**`Calibration Ladder Wiring v1` remains deferred** until source_substitution ships and starts producing artifacts with populated `ConfidencePermission` and `PosturePermission` against real outcomes.

---

## 11. risks and open questions

- **Trial scrambling defeats free-trial verification.** Anyone running the verification protocol on a free trial sees real schema and scrambled values. The slice cannot be closed on a free trial alone. Either upgrade to a paid plan or request an unscrambled trial from sales explicitly.
- **Tier identity is unconfirmed.** Four named betting-data products exist on SportsDataIO's product portfolio. `BettingSplit` is documented but not bound to a specific tier in the public copy. Phase A must include a sales question: "Which subscription tier includes `BettingSplit` payloads for NBA, NFL, and MLB?" If splits are a paid add-on to the base Odds API, the cost ceiling moves and that affects the slice budget.
- **The "sportsbook discretion" caveat is the same risk that produced this entire chain.** SportsDataIO explicitly disclaims that splits availability depends on sportsbook willingness. If the NBA playoff blackout is market-wide (not provider-specific), source_substitution will fail regardless of provider choice, and the next rung becomes fidelity_downgrade or accepting `unavailable_with_reason` during playoffs.
- **Endpoint name is inferred, not documented.** Whether `BettingSplit` returns inline via `BettingMarketsByGameID?include=...` or via a separate `BettingSplitsByGameID` endpoint is not in the public docs reviewed here. Phase A must resolve this; both possibilities are covered by the verification commands in section 4.
- **Sportsbook scope is undisclosed.** "25+ books" is a marketing number. Which books actually publish splits to SportsDataIO — and which subset publishes during NBA playoffs specifically — is not documented. Phase A should record a distinct-sportsbook count in the verification log.
- **Rate-limit and freshness behavior is undocumented.** SportsDataIO's general rate-limit posture per tier was not visible in public copy. Phase A must record observed cadence so the production cache TTL can be sized correctly.
- **Pricing remains undisclosed publicly.** Slice budget for v1 cannot be finalized without sales contact.

---

## 12. final decision

**Proceed conditionally to Sharp/Public Alternate Provider v1.** Phase A (paid-key or unscrambled-trial verification, ≤ 1 developer-day) is a hard gate. Phase B (client implementation) does not start until Phase A returns populated `BetPercentage` and `MoneyPercentage` for at least one current NBA playoff game.

If Phase A fails, reject SportsDataIO and re-scope to **Sharp/Public Fidelity Downgrade v1** (extend the existing ActionNetwork client to surface a degraded directional signal from the still-available fields documented in the verification-results doc). VSiN remains the legal-cleared fallback if a sponsor wants to invest in formal licensing or scrape-licensing review.

---

## references

- discovery doc: `sharp-public-alternate-provider-discovery-v1.md`
- prior verification: `sharp-public-provider-verification-results.md`
- prior investigation: `sharp-public-provider-investigation-v1.md`
- ladder doctrine: `02 Platform/architecture/cognitive-factory/signal-fallback-ladder.md`
- target contract sketch: `sharp-public-alternate-provider-discovery-v1.md` §7

external pages reviewed for this verification (no runtime calls made):

- sportsdata.io/developers/api-documentation/nba
- sportsdata.io/developers/workflow-guide/nba
- sportsdata.io/developers/data-dictionary/nba (data dictionary confirmed `BettingSplit` object and `BetPercentage` / `MoneyPercentage` fields)
- sportsdata.io/live-odds-api
- sportsdata.io/free-trial (confirmed trial data is scrambled by design)
- sportsdata.io/help/betting-data-integration-guide
- support.sportsdata.io/hc/en-us/articles/4404845466519 (403 on this fetch; same content is also at the main-site help URL above)
