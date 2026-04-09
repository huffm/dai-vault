# schema: niche config

## purpose
defines the structure of the `config` object on a tenant record. niche config is what makes a tenant's workflow run differently from another tenant on the same niche. it is not platform logic — it is the per-tenant parameterization of a niche assembly line.

## principle
platform agents are generic. niche config is the set of values injected at runtime that makes them niche-specific: which data sources to call, which signals to score, what thresholds to apply, what output format to use.

---

## sports-analytics config

| field | type | description |
|---|---|---|
| `sports` | string[] | list of active sports (e.g. `["NFL"]`) |
| `delivery_timing_hours_before` | number | how many hours before kickoff to trigger the brief |
| `line_move_alert_threshold` | number | point spread movement that triggers a re-alert (e.g. `2`) |
| `signal_weights` | object | optional overrides for per-signal weight in aggregate score |

---

## crypto-analytics config

| field | type | description |
|---|---|---|
| `watchlist` | string[] | asset tickers to cover (e.g. `["BTC", "ETH"]`) |
| `max_watchlist_size` | number | enforced by tier (free=1, starter=5, pro=unlimited) |
| `delivery_time_utc` | string | cron-style time for daily brief (e.g. `"06:00"`) |
| `funding_rate_alert_threshold_high` | number | upper funding rate threshold for alert (e.g. `0.001`) |
| `funding_rate_alert_threshold_low` | number | lower funding rate threshold for alert (e.g. `-0.0005`) |

---

## stock-analytics config

| field | type | description |
|---|---|---|
| `watchlist` | string[] | ticker symbols to cover (e.g. `["AAPL", "NVDA"]`) |
| `max_watchlist_size` | number | enforced by tier (free=3, starter=10, pro=50) |
| `earnings_brief_hours_before` | number | hours before earnings to trigger brief (default: `48`) |
| `weekly_brief_day` | string | day to trigger weekly brief (default: `"friday"`) |

---

## kalshi-analytics config

| field | type | description |
|---|---|---|
| `market_categories` | string[] | categories to track (e.g. `["economic", "political"]`) |
| `pinned_markets` | string[] | specific market slugs the tenant has pinned (pro only) |
| `price_move_alert_threshold` | number | point movement that triggers a price alert (e.g. `5`) |
| `max_pinned_markets` | number | enforced by tier |

---

## rules
- config is validated at tenant creation and on update
- tier-enforced limits (watchlist size, pinned markets) are checked against the tenant's active stripe subscription at runtime — do not hard-code tier logic into config objects
- signal_weights overrides are optional; if absent, defaults defined in the niche workflow apply
- niche config does not contain api keys or credentials — those belong in platform secrets management
