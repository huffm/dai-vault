# sharp/public provider investigation v1

**date:** 2026-05-11
**status:** discovery only — no provider integration, no production change, no Line Movement Proxy work.
**ladder rung under investigation:** exact_recovery (highest rung) for the missing `sharp_public` primary signal.
**outcome:** exact_recovery is **plausibly viable** via a low-risk bookId expansion. recommend a small verification protocol first, then commit to one of two follow-on slices.

---

## 1. summary

Every recent NBA playoff run shows `sharp_public` missing with reason `provider_returned_empty_odds`. Four artifacts from 2026-05-09 and 2026-05-10 confirm the pattern. The question this slice answers: *can we recover the same signal from the same provider before we move to source_substitution or adjacent_proxy?*

Findings:

1. The current client queries a single sportsbook (`bookIds=15`) against a single endpoint. The provider's own web frontend queries many books in parallel.
2. The current telemetry conflates two different failure modes under one reason string (`provider_returned_empty_odds`) — we cannot tell from the artifact whether the `odds` property is absent on the game or the `odds` array is empty for bookId=15 specifically.
3. ActionNetwork's web UI displays sharp/public percentages for many of these same playoff games, which is a strong external signal that *some* book in the catalog has the data even when bookId=15 returns nothing.
4. The client already iterates multiple entries inside the `books_odds` / `odds` array and picks the first with populated percentages. Expanding `bookIds=` is a natural extension of that pattern — no new abstraction required.

Decision: **exact_recovery is the most likely cheapest win**. Run a 30–60 minute manual verification (curl from a dev shell) before committing. If verification passes, recommend **Sharp/Public Schema Repair v1**. If verification fails for all reasonable book IDs, recommend **Sharp/Public Alternate Provider v1**.

Line Movement Proxy v1 remains deferred — it is `adjacent_proxy`, not equivalent.

---

## 2. current sharp_public source path

**File:** `dai/platform/dotnet/DevCore.Api/Sports/ActionNetworkClient.cs`

**Path:**

```
SportsRetriever.RetrieveAsync (nfl/ncaaf/nba/ncaamb)
  → ActionNetworkClient.GetSharpPublicDataAsync(competition, homeTeam, awayTeam, gameDate, ct)
    → cache check (15-minute TTL, key: sharpPublic:{sportKey}:{home}:{away}:{date})
    → ActionNetworkClient.FetchAsync
      → GET https://api.actionnetwork.com/web/v1/scoreboard/{sportKey}?date=YYYYMMDD&period=full&bookIds=15
      → ActionNetworkClient.ParseMatch (find game by team name, then percentages from books_odds[] or odds[])
    → SharpPublicLookupResult { Context, Status, MissingReason, Detail }
SharpPublicLookupResult flows into SportsRetrievalOutput → SignalAvailabilityRecord → SignalQualityEvaluator → SignalFollowUpEvaluator → OutputJson.
```

**Cache:** `IMemoryCache`, 15-minute absolute expiration. Reads are cached even for `missing` results, which is desirable (we do not want to hammer the unofficial API on miss).

**Auth:** none. The client uses the publicly accessible `api.actionnetwork.com` host with a `Mozilla/5.0 (compatible; DevCore/1.0)` User-Agent. No API key, no Authorization header. This is the same endpoint the public web frontend hits.

---

## 3. current request shape

```
Method:  GET
Host:    api.actionnetwork.com
Path:    /web/v1/scoreboard/{sportKey}
Query:   date={YYYYMMDD}&period=full&bookIds=15
Headers: User-Agent: Mozilla/5.0 (compatible; DevCore/1.0)
Timeout: 10 seconds
```

`sportKey` mapping (in `ActionNetworkClient._sportKeys`):

| competition | sportKey |
|---|---|
| nfl | nfl |
| ncaaf | ncaaf |
| nba | nba |
| ncaamb | ncaab |
| mlb | (excluded) |

**Single book pinned:** `bookIds=15` is the only book queried today. The 2026-05-08 schema-repair notes record that without `bookIds`, the `odds` array is always empty. That note proved out at the time, but it was validated against regular-season games. Whether the same statement holds during playoffs is the open question this slice surfaces.

---

## 4. current response fields used

The client tolerates two coexisting schemas. Both are observed in production.

**Schema A — "books_odds" array (older nfl/ncaaf shape):**

```
games[i] = {
  away_team: { full_name, alias },
  home_team: { full_name, alias },
  start_time,
  books_odds: [
    { book_id, spread_home_public, spread_away_public,
      spread_home_money, spread_away_money, updated_at }
  ]
}
```

**Schema B — "teams" array + "odds" array (current nba/ncaamb shape):**

```
games[i] = {
  away_team_id, home_team_id,
  teams: [ { id, full_name, alias } ],
  start_time,
  odds: [
    { book_id, spread_home_public, spread_away_public,
      spread_home_money, spread_away_money, updated_at }
  ]
}
```

The client tries A first, falls back to B. It iterates the chosen array and accepts the first entry where all four percentage fields are non-null integers.

**Telemetry collapse point:** the missing-reason vocabulary today is:

| reason | trigger |
|---|---|
| `provider_returned_empty_odds` | `odds`/`books_odds` property absent **OR** present but empty array |
| `provider_returned_null_pct` | entries present but all percentages JSON-null |
| `no_matching_game` | no game matched the team names |
| `provider_error` | HTTP non-success or parse exception |
| `invalid_date` | game date could not be parsed |
| `unsupported_competition` | competition not in the supported set |

The first row is the one that fires on every recent NBA playoff miss. Two physically different conditions land in the same string. That is the first concrete diagnostic gap.

---

## 5. known missing pattern from calibration artifacts

Inspected all 14 calibration artifacts under `04 Products/sports-v1/calibration/artifacts/`. `sharp_public` appears as a structured `signalAvailability` entry in 4 of them. All 4 are missing with the same reason.

| date | competition | matchup id | status | reason |
|---|---|---|---|---|
| 2026-05-09 08:17 | nba | f5bc433e | missing | provider_returned_empty_odds |
| 2026-05-09 15:25 | nba | fcbc433e | missing | provider_returned_empty_odds |
| 2026-05-09 15:54 | nba | 00bd433e | missing | provider_returned_empty_odds |
| 2026-05-10 19:13 | nba | 04bd433e | missing | provider_returned_empty_odds |

Older artifacts (2026-05-07, 2026-05-08-1059) do not include `sharp_public` in their availability records at all — those runs predate Signal Availability Diagnostics v1 enrichment for sharp_public. No artifact in the corpus shows `sharp_public` as `grounded`.

**Pattern observations:**

- 100% of observed misses are NBA.
- 100% fall within the 2026 NBA playoff window (May 9–10).
- 100% report the same reason — but as noted in section 4, that reason collapses two distinct conditions.
- 0 observations of `provider_returned_null_pct` in production. Either the playoff response truly has no entries for bookId=15, or the property is absent altogether. We cannot tell from current telemetry.
- 0 observations of `no_matching_game`, `provider_error`, `invalid_date`, or `unsupported_competition` for sharp_public in the current corpus — team-name matching, network, and date parsing are not the failure mode.

The pattern is consistent enough to act on. The corpus is small enough that we should run the verification protocol below before committing to an implementation slice.

---

## 6. exact recovery options investigated

Six exact-recovery levers were considered for the same signal from the same provider before reaching for source_substitution.

### 6a. Expand bookIds beyond 15 (most likely fix)

The `bookIds` query parameter accepts a comma-separated list. The ActionNetwork web frontend itself requests many books in parallel and renders whichever has data. The current client pins to bookId=15 (FanDuel). When bookId=15 returns no entries for a given game, the entire result is reported missing — even when other books in the same response would have populated percentages.

Reverse-engineered book IDs from the public web frontend at time of last check:

| book_id | sportsbook (approximate, may rotate) |
|---|---|
| 15 | FanDuel |
| 30 | Caesars |
| 68 | PointsBet |
| 69 | DraftKings |
| 75 | BetMGM |
| 76 | William Hill |
| 123 | WynnBet |
| 283 | Bet365 |

The cheapest exact-recovery attempt: query `bookIds=15,30,68,69,75,76,123,283`, then continue the existing "first entry with all four populated" loop — across books, not just across snapshots within a single book.

**Risk:** the unofficial API may rate-limit larger queries. The 15-minute cache mitigates this. Even with the expanded list, the request body and response shape are unchanged from the client's perspective.

### 6b. Distinguish empty-array vs missing-property in the existing reason

Currently `provider_returned_empty_odds` covers both. Splitting into `provider_returned_no_odds_property` and `provider_returned_empty_odds_array` makes the playoff failure mode legible without changing what the analyzer or evaluator does. This is a small calibration-only change.

**Recommendation:** include this split in the Sharp/Public Schema Repair v1 slice. Do not ship it as a standalone change — it is one line of value on its own and the bigger fix carries it cleanly.

### 6c. Use period=game or period=2H instead of period=full

The `period` parameter controls whether the spread is full-game or a sub-period. Sharp/public splits during playoffs may be reported only against sub-period markets if the operator suspends pre-game lines for integrity reasons. **Low likelihood**, but cheap to verify.

### 6d. Try `/web/v2/scoreboard` if it exists

ActionNetwork has historically run v1 and v2 endpoints in parallel during transitions. If v2 exists and the playoff data lives there, swapping the path is an exact-recovery move (same provider, same signal, different surface). **Unknown likelihood** — needs HTTP verification.

### 6e. Use `/games/{game_id}` per-game endpoint

If the scoreboard endpoint truncates per-game odds during playoffs to keep the payload small, a per-game endpoint may carry the full odds. **Medium likelihood**. This would be a slightly bigger client change (two-stage fetch: scoreboard for game IDs, then per-game for odds) but still same provider, same signal.

### 6f. Retry/normalize after a delay

The current cache caches misses for 15 minutes. Sharp/public percentages publish later in the day as volume builds. The current loop already accepts the first entry with populated percentages within the response, but it does not retry the call later. Retrying after delay is not "exact recovery from the response we have," it is "exact recovery from the same source on a later request." This belongs in the cache TTL conversation, not in the schema repair.

**Recommendation:** consider shortening the negative-cache TTL (e.g. 5 minutes for misses, 30 minutes for hits) as part of the schema repair, so a later run on the same matchup can pick up newly-published percentages.

---

## 7. findings per option

| option | likelihood | cost | risk | recommended in scope |
|---|---|---|---|---|
| 6a expand bookIds | **high** | small (one-line URL change + loop already exists across entries) | minor rate-limit risk, mitigated by cache | **yes** — primary recommendation |
| 6b split reason vocabulary | n/a (diagnostic) | very small | none | yes — bundle with 6a |
| 6c period=game / period=2H | low | small (separate fetch) | adds complexity for marginal gain | no — defer unless 6a fails |
| 6d v2 scoreboard | unknown | medium (different parser) | API contract drift | defer pending HTTP verification |
| 6e per-game endpoint | medium | medium (two-stage fetch) | adds latency, doubles cache surface | defer pending 6a outcome |
| 6f retry after delay / split negative TTL | medium | small | minor — already cached | bundle with 6a if the data is genuinely late-publishing |

**Critical unknown that gates the recommendation:** we have not yet *verified* that bookIds beyond 15 carry populated percentages during the 2026 NBA playoffs. The recommendation is **conditional on verification**.

---

## 8. verification protocol (manual, do not wire)

Run these from a developer shell against the live API before committing to Sharp/Public Schema Repair v1. No production code change required to verify.

**Step 1 — confirm the failure mode for bookId=15:**

```bash
curl -sS -A "Mozilla/5.0 (compatible; DevCore/1.0)" \
  "https://api.actionnetwork.com/web/v1/scoreboard/nba?date=20260510&period=full&bookIds=15" \
  | jq '.games[] | {id: .id, home_team_id, away_team_id, has_odds: (.odds // [] | length)}'
```

Expect: most playoff games show `has_odds: 0`. This is the current failure path.

**Step 2 — try the expanded book list:**

```bash
curl -sS -A "Mozilla/5.0 (compatible; DevCore/1.0)" \
  "https://api.actionnetwork.com/web/v1/scoreboard/nba?date=20260510&period=full&bookIds=15,30,68,69,75,76,123,283" \
  | jq '.games[] | {id, has_odds: (.odds // [] | length), books: [(.odds // [])[] | {book_id, hp: .spread_home_public, ap: .spread_away_public}]}'
```

Expect (hypothesis): at least one playoff game shows non-zero `has_odds` with at least one book having non-null `spread_home_public` and `spread_away_public`.

**Step 3 — pick one failing game and try alternates:**

For the Spurs vs Timberwolves game (`agentRunId 04bd433e`, game date 2026-05-10), repeat step 2 with:

- `period=game` — does sub-period have what full does not?
- swap path to `/web/v2/scoreboard/nba` — does v2 exist and carry data?
- if step 2 succeeds with one specific book, use that book's per-game endpoint (`/games/{id}/odds?book_id=X`) to compare.

**Decision rule from verification:**

- Step 2 finds populated percentages from any book → **exact_recovery viable** → recommend **Sharp/Public Schema Repair v1**. Slice scope: expand `bookIds`, split the missing-reason vocabulary, optionally shorten negative TTL. No new provider, no new auth.
- Step 2 fails across all books AND step 3 alternates fail → **exact_recovery not viable** → recommend **Sharp/Public Alternate Provider v1** as source_substitution. Begin scoping VSiN, BetIQ, or a paid feed.

The verification is one developer-hour of work. It is the cheapest decision-quality move in this entire investigation.

---

## 9. recommendation

**Run verification first.** Section 8 above is the entire protocol. Total time: under one hour.

**If verification succeeds** (likely):

> **Sharp/Public Schema Repair v1** — exact_recovery slice. Expand `ActionNetworkClient` to query a configurable list of `bookIds`, iterate across books and within-book entries, and accept the first entry with all four percentage fields populated. Split `provider_returned_empty_odds` into two distinct reasons so future calibration artifacts tell us which branch fired. Optionally shorten negative cache TTL. No new provider, no new auth, no contract change for downstream consumers.

**If verification fails:**

> **Sharp/Public Alternate Provider v1** — source_substitution slice. Scope a second sharp/public source. Treat the result as `source_substitution / near` on the fallback ladder. Higher cost, but real confidence-permission upside (`confidence_mostly_preserved`).

Both options remain above `adjacent_proxy / line_movement` on the ladder. Both must be exhausted before reaching for adjacent proxies.

---

## 10. next slice

The next slice is determined by the verification protocol outcome.

**Default (verification likely succeeds):**

- **Sharp/Public Schema Repair v1**

**Contingency:**

- **Sharp/Public Alternate Provider v1**

**Explicitly deferred (still):**

- **Line Movement Proxy v1** — adjacent_proxy / adjacent equivalence. Not a substitute for sharp_public. Land only after both higher rungs are tried or ruled out.

**Wiring slice (later, scoped against outcome data):**

- **Calibration Ladder Wiring v1** — feed `ConfidencePermission` and `PosturePermission` into `SportsEvaluator` so the ladder stops being purely observational.

---

## 11. risks and open questions

- **API drift risk.** ActionNetwork is unofficial. The expanded `bookIds` request could rate-limit, return a different shape, or rotate book IDs without notice. The existing defensive parsing tolerates field absence; the same posture should apply to the expanded request. The 15-minute cache is the primary mitigation.
- **Book-ID rotation.** The book_id values listed in section 6a are reverse-engineered. They may shift. The Sharp/Public Schema Repair v1 slice should pull `bookIds` from configuration, not a code constant, so the list can be updated without a redeploy.
- **Telemetry conflation persists today.** Until the missing-reason vocabulary is split (option 6b), every miss looks the same in the artifact. Reading this investigation requires keeping in mind that a single reason value masks two physical conditions.
- **Verification is provider-state-dependent.** The verification protocol must be run **against the live provider during a similar playoff window** to give a clean answer. Running it during the regular season would not exercise the failing path.
- **Cache impact on verification.** If a developer runs the verification immediately after a missed retrieval, the 15-minute negative cache may interfere. Manual curl bypasses the cache; the recommendation still stands.
- **Possible legal/compliance angle.** Some sportsbooks reduce or suppress public splits during playoff games for integrity reasons. If verification reveals that *every* book legitimately has no data on those games, the gap is provider-side and only source_substitution to a non-sportsbook feed (e.g. an integrity-monitoring service) will fix it. That is a strictly bigger and more expensive slice.
- **No production code was changed in this slice.** All findings here are derived from artifacts and source reading. The diagnostic improvement listed in option 6b is deliberately bundled with Sharp/Public Schema Repair v1 to avoid shipping a one-line cosmetic change in isolation.

---

## references

- `dai/platform/dotnet/DevCore.Api/Sports/ActionNetworkClient.cs` — current client implementation.
- `dai/platform/dotnet/DevCore.Api.Tests/Sports/ActionNetworkClientTests.cs` — confirms parsing fixtures and the two-branch collapse for `provider_returned_empty_odds`.
- `dai-vault/02 Platform/architecture/cognitive-factory/signal-fallback-ladder.md` — the ladder this investigation walks top-down.
- `dai-vault/04 Products/sports-v1/calibration/artifacts/20260510-1913-nba-04bd433e.json` — representative failing artifact.
- `dai-vault/04 Products/sports-v1/current-sports-analysis-flow.md` — pipeline detail, including the 2026-05-08 schema-repair notes that established `bookIds=15`.
- `dai/.claude/skills/dai-signal-follow-up-diagnostics/rules.md` — the rule that requires walking the ladder top-down before recommending adjacent proxies.
