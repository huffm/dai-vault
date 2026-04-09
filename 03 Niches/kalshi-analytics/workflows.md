# workflows: kalshi-analytics

## v1 core workflow: pre-event brief

### trigger
runs 24 hours before any major scheduled economic release (CPI day, NFP Friday, FOMC announcement). this is the highest-value moment — traders are actively sizing positions. a daily sweep of all markets is secondary to nailing event-driven coverage.

### steps
1. fetch active kalshi markets via kalshi api
2. filter to high-liquidity markets in covered categories
3. fetch current prices and 24-hour price movement
4. fetch external reference probabilities (polymarket, metaculus, model estimates)
5. score each market: mispriced high / fairly priced / mispriced low
6. rank by confidence and magnitude of divergence
7. run summarizer agent to produce structured daily brief
8. deliver to tenant

## brief output format
- date and market count header
- top flags: markets with highest-confidence mispricing
- full watchlist table: market name, current price, reference probability, divergence score
- narrative summary: 3-5 sentences on the overall market picture
- upcoming events: scheduled resolutions in next 48 hours

## v1 secondary workflow: daily market sweep

### trigger
runs once daily at 07:00 UTC as a background scan of all active kalshi markets. surfaces any contract outside of scheduled events that shows a significant price/reference divergence.

## v1 tertiary workflow: price movement alert

### trigger
price moves more than N points on a watched market within a single day.

### steps
1. detect price movement threshold breach
2. fetch current context for that market
3. produce a single-market alert brief
4. push to tenant immediately

## agent roles used
- collector: pulls kalshi market data and external reference probabilities
- evaluator: calculates price divergence and confidence score per market
- synthesizer: writes brief narrative from scored markets
- compliance: confirms output does not constitute regulated financial advice or violate scope constraints
- delivery: formats and dispatches the brief to the tenant's configured channel

## what this workflow does not do
- does not place trades on kalshi
- does not track tenant positions or p&l
- does not monitor markets in real-time (v1 is daily + event-triggered)
