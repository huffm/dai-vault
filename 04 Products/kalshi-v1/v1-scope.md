# v1 scope: kalshi-v1

## in scope

### pre-event brief (primary workflow)
- triggers 24 hours before each major scheduled economic release (CPI, NFP, FOMC)
- covers per relevant Kalshi market:
  - current yes/no price
  - 24-hour price movement
  - volume vs recent average
  - CME futures implied probability (FOMC markets)
  - economic consensus estimate vs Kalshi resolution threshold (CPI/NFP markets)
  - Polymarket equivalent price if available
  - Metaculus community estimate if available
  - divergence score: mispriced high / fairly priced / mispriced low
  - confidence flag based on number of reference signals available
  - days to resolution
  - 2-3 sentence context note

### daily market sweep (secondary workflow)
- runs once daily at 07:00 UTC
- scans all active Kalshi markets in covered categories
- surfaces any contract showing a significant price/reference divergence outside of scheduled event windows

### price movement alert (tertiary workflow, pro only)
- fires when a watched Kalshi market moves more than N points in a single day
- delivers a short single-market context note

### delivery
- email (starter), slack/webhook (pro)

### tiers
- free: top 5 flagged markets from daily sweep, same-day delivery
- starter: full daily sweep + pre-event briefs, email delivery
- pro: all of the above + price movement alerts + custom watchlist pinning

## out of scope for v1
- political markets (expand post-validation)
- weather markets (expand after political)
- calibration history scoring (requires 3+ months of resolved market data — v2)
- intraday real-time monitoring

## key decisions for v1 build
- data sources: Kalshi API (markets/prices), CMEFedWatch (rate decisions), Econoday or Briefing.com (economic calendar/consensus), Polymarket API (cross-reference), Metaculus API (forecasting aggregate), Open-Meteo (weather, when expanded)
- economic calendar scan runs daily to detect upcoming events within 24-hour trigger window

## definition of done for v1
- pre-event brief produces correctly for CPI and NFP test events
- divergence scoring fires accurately against known reference probabilities
- 50 paying users within 60 days of launch — if not met, reassess as standalone product
