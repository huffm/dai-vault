# sharp/public provider verification results

**date:** 2026-05-11
**status:** verification of `sharp-public-provider-investigation-v1.md` section 8 — discovery only, no production code changed.
**outcome:** exact_recovery is **NOT viable** at the public ActionNetwork endpoint for current NBA playoff games. Promotes the next slice to **Sharp/Public Alternate Provider v1** (source_substitution).

---

## what was tested

The discovery doc framed three concrete steps. All three were executed against the live `api.actionnetwork.com` host with the same User-Agent the production client uses. The Spurs at Timberwolves matchup (`agentRunId 04bd433e`, gameId `290435`, game date `2026-05-10`) was the documented anchor; checks were broadened to four dates and an alternate endpoint to rule out single-game flukes.

| matchup | game date | gameId | status |
|---|---|---|---|
| Spurs at Timberwolves | 2026-05-10 | 290435 | completed |
| Knicks at 76ers | 2026-05-10 | 290492 | completed |
| Cavaliers at Pistons | 2026-05-09 | 290579 | completed |
| Thunder at Lakers | 2026-05-09 | 290457 | completed |
| Cavaliers at Pistons | 2026-05-11 | 290580 | scheduled |
| Thunder at Lakers | 2026-05-11 | 290458 | scheduled |
| Knicks at 76ers | 2026-05-12 | 290493 | cancelled |
| Timberwolves at Spurs | 2026-05-12 | 290436 | scheduled |
| Cavaliers at Pistons | 2026-05-13 | 290581 | scheduled |
| Thunder at Lakers | 2026-05-13 | 290459 | scheduled |

All ten games carry `type: post` (playoff) and `season: 2025` in the ActionNetwork response.

---

## results

### 1. bookIds=15 baseline (current production query)

| query | result |
|---|---|
| `GET /web/v1/scoreboard/nba?date=20260510&period=full&bookIds=15` | both games returned `odds: []` (empty array). Confirms the production failure mode. |

### 2. expanded bookIds

| query | result |
|---|---|
| `GET /web/v1/scoreboard/nba?date=YYYYMMDD&period=full&bookIds=15,30,68,69,75,76,123,283` (4 dates, 8 games) | every game returned `odds: []`. Response bytes were byte-identical to the bookIds=15 response (`12361 bytes` on the anchor date). The endpoint appears to ignore book additions when the `period=full` filter is set on playoff games. |

### 3. drop `period=full` (uncovers the underlying odds array)

| query | result |
|---|---|
| `GET /web/v1/scoreboard/nba?date=20260510&bookIds=15` | both completed playoff games returned **22 odds entries each**, spanning **book_ids `1, 15, 21, 30`**. **All 22 entries had `spread_home_public: null`, `spread_away_public: null`, `spread_home_money: null`, `spread_away_money: null`, `ml_home_public: null`, `ml_away_public: null`, `total_over_public: null`, `total_under_public: null`** and the corresponding `_money` fields. Spread/total/moneyline prices were populated; only the public/sharp split fields were null. |

### 4. v2 scoreboard endpoint

| query | result |
|---|---|
| `GET /web/v2/scoreboard/nba?date=20260510&period=full&bookIds=...` | HTTP 200, 54023 bytes. Different schema (game-level `core_id` key). `odds: []` on every game. No public/sharp split fields surfaced. Not a viable exact_recovery path. |

### 5. per-game endpoint

| query | result |
|---|---|
| `GET /web/v1/games/290435?bookIds=...` | HTTP **404** — the per-game endpoint at this path does not exist or requires authentication. |

### 6. methodology sanity check — regular season

| query | result |
|---|---|
| `GET /web/v1/scoreboard/nba?date=20250120` | 8 of 8 regular-season games (type=reg) returned populated `spread_home_public`, `spread_away_public`, `spread_home_money`, `spread_away_money` fields at `book_id=15`. Example: Charlotte Hornets at Dallas Mavericks → 32/68 public split, 36/64 money split. **Confirms the methodology works** — the playoff failure is a real provider-side data gap, not a query bug. |

---

## commands run (verbatim)

```bash
# 1. baseline (production query shape)
curl -sS -A "Mozilla/5.0 (compatible; DevCore/1.0)" --max-time 15 \
  "https://api.actionnetwork.com/web/v1/scoreboard/nba?date=20260510&period=full&bookIds=15"

# 2. expanded bookIds (the hypothesis)
curl -sS -A "Mozilla/5.0 (compatible; DevCore/1.0)" --max-time 15 \
  "https://api.actionnetwork.com/web/v1/scoreboard/nba?date=20260510&period=full&bookIds=15,30,68,69,75,76,123,283"

# 3. expanded across four dates (rule out single-game flukes)
for d in 20260509 20260511 20260512 20260513; do
  curl -sS -A "Mozilla/5.0 (compatible; DevCore/1.0)" --max-time 15 \
    "https://api.actionnetwork.com/web/v1/scoreboard/nba?date=$d&period=full&bookIds=15,30,68,69,75,76,123,283"
done

# 4. drop period=full to uncover the underlying odds array
curl -sS -A "Mozilla/5.0 (compatible; DevCore/1.0)" --max-time 15 \
  "https://api.actionnetwork.com/web/v1/scoreboard/nba?date=20260510&bookIds=15"

# 5. v2 endpoint
curl -sS -A "Mozilla/5.0 (compatible; DevCore/1.0)" --max-time 15 \
  "https://api.actionnetwork.com/web/v2/scoreboard/nba?date=20260510&period=full&bookIds=15,30,68,69,75,76,123,283"

# 6. per-game endpoint
curl -sS -A "Mozilla/5.0 (compatible; DevCore/1.0)" --max-time 15 \
  "https://api.actionnetwork.com/web/v1/games/290435?bookIds=15,30,68,69,75,76,123,283"

# 7. regular-season sanity check
curl -sS -A "Mozilla/5.0 (compatible; DevCore/1.0)" --max-time 15 \
  "https://api.actionnetwork.com/web/v1/scoreboard/nba?date=20250120"
```

Raw responses are intentionally not committed. They are inspectable on demand via the commands above.

---

## conclusion

**Exact recovery is NOT viable** at the public ActionNetwork endpoint for the current NBA playoff window.

Three separate angles confirm this:

1. Expanded bookIds returned byte-identical responses to bookIds=15 — the endpoint genuinely has no additional book data when `period=full` is set.
2. Dropping `period=full` did surface 22 odds entries per game across 4 distinct book_ids — but every percentage field was null on every entry. The data is absent at the source, not filtered out client-side.
3. v2 scoreboard returned a different schema with `odds: []` for every game. The per-game endpoint returned 404.

The regular-season sanity check at `2025-01-20` confirmed that the methodology, parsing, and field names are correct — populated splits flow through normally when the provider has data. The 2026 NBA playoff gap is provider-side.

**Secondary diagnostic finding:** the current production query (`period=full&bookIds=15`) returns `provider_returned_empty_odds` because the period filter strips the odds array. Without `period=full`, the same games return 22 entries with null percentages — the same underlying reality, but a different missing-reason classification. The current telemetry vocabulary is not wrong, but it is misleading: it implies "no entries at all" when in fact entries exist but have null splits. This is a small calibration-quality observation, not a recovery path.

---

## recommended next slice

**Sharp/Public Alternate Provider v1** (source_substitution).

Scope a second sharp/public source. Candidates to investigate (in order of likely cost / access ceremony):

1. **VSiN public splits** — sportsbook-adjacent, sometimes free with attribution.
2. **BetIQ / Sports Insights** — paid API, used by several derivative products.
3. **Sportradar / Genius Sports Integrity feeds** — enterprise tier; only worth scoping if playoff coverage is non-negotiable.
4. **Bet The Number / Don Best** — paid splits aggregator; primarily a B2B feed.

This recommendation respects the fallback ladder. `Sharp/Public Schema Repair v1` is no longer the right slice — there is no schema repair that can recover null fields from the provider.

**Still deferred (per the ladder):**

- **Line Movement Proxy v1** — `adjacent_proxy / adjacent`. Cannot be used as a substitute for missing `sharp_public`. Land only after `source_substitution` is shipped or definitively ruled out.

**Optional micro-cleanup that can land alongside the source_substitution slice (not a slice on its own):**

- Drop `period=full` from the production query and rely on the existing "first entry with all four populated" loop to filter. This makes the missing-reason classification report `provider_returned_null_pct` when entries exist but splits are null — closer to the real failure mode. The behavior change is observational only; no analyzer or evaluator output shifts.

---

## risks

- **Unofficial endpoint.** The verification was run against a public endpoint with no auth. ActionNetwork can rotate book IDs, change parameter behavior (the `period=full` filter behavior observed here is itself undocumented), or rate-limit at any time. Any source_substitution candidate should be evaluated on contract stability as well as data quality.
- **Verification is provider-state-dependent.** Results reflect 2026-05-11 state. If ActionNetwork later starts publishing splits for completed playoff games, this verification will need to be re-run. The methodology section gives a reusable check (run command 1 against any current playoff date; if `has_odds: 0` persists, the gap is still active).
- **Schema drift not observed.** Two coexisting response schemas (`books_odds[]` and `teams[]+odds[]`) were both anticipated by the current parser. No new schema variant surfaced during verification.
- **No production behavior was changed.** The diagnostic observation about `period=full` is documented as a micro-cleanup for the next slice, not shipped on its own.
- **Source substitution adds dependency surface.** Wiring a second provider increases auth, billing, schema mapping, and failure-mode count. The Sharp/Public Alternate Provider v1 slice scope must include a clean "which source did this percentage come from" provenance field on the artifact so calibration can grade providers independently over time.

---

## references

- `sharp-public-provider-investigation-v1.md` — the discovery doc this slice verifies.
- `dai/platform/dotnet/DevCore.Api/Sports/ActionNetworkClient.cs` — the production client whose behavior was under test.
- `dai-vault/02 Platform/architecture/cognitive-factory/signal-fallback-ladder.md` — the ladder this verification walks.
- `dai-vault/04 Products/sports-v1/calibration/artifacts/20260510-1913-nba-04bd433e.json` — the documented missing-signal artifact this verification anchored against.
