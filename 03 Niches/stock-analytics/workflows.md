# workflows: stock-analytics

## v1 core workflow: earnings brief

### trigger
scheduled job runs 48 hours before each earnings event for any stock on the tenant's watchlist.

### steps
1. pull earnings calendar for tenant watchlist
2. fetch price, volume, and technical data
3. fetch analyst estimate revisions and expected move from options market
4. fetch insider activity from sec edgar (form 4 filings)
5. fetch sector performance relative to sp500
6. score each signal category
7. run summarizer agent to produce structured pre-earnings brief
8. deliver brief to tenant

## v1 secondary workflow: weekly watchlist brief

### trigger
friday after market close (4:30 PM ET). captures the week's moves while fresh. delivered sunday evening so tenants have it before monday open.

### steps
1. fetch current price and technical state for all watchlist stocks
2. score each signal category at current state
3. produce one-page watchlist summary with flags for any stock showing signal shift
4. deliver to tenant

## brief output format
- stock header: ticker, current price, upcoming catalyst if any
- signal table: each category scored with flag
- narrative: 3-5 sentences on the current setup
- watch flag: call out if anything has changed since last brief

## agent roles used
- collector: pulls data per source
- evaluator: scores each signal category
- synthesizer: writes narrative from scored signals
- compliance: confirms output does not constitute regulated investment advice or violate scope constraints
- delivery: formats and dispatches the brief to the tenant's configured channel

## what this workflow does not do
- does not track portfolio positions or p&l
- does not place orders
- does not run intraday in v1
