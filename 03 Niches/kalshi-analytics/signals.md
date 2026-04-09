# signals: kalshi-analytics

## signal categories

### market pricing
- current yes/no price vs model probability estimate
- price movement in past 24 hours
- volume spike relative to recent average

### resolution context
- days to resolution
- known scheduled events that could resolve the market (fed meeting, jobs report, election)
- recent news relevant to the market topic

### calibration history
- how accurate similar past markets have been relative to their prices
- typical resolution pattern for this market category (economic, weather, political)
- note: this requires accumulating resolved market data over time. treat as v2 until at least 3 months of data is available. do not fake it in v1.

### external probability references
- polymarket or manifold equivalent market price (if available)
- forecasting aggregator estimate (e.g. metaculus, aggregate polling)
- model-based estimate where calculable (e.g. weather forecast for weather markets)

## signal scoring (v1 approach)
each market scored as: looks mispriced high / fairly priced / looks mispriced low
based on delta between market price and available reference probability.
confidence flag based on how many reference signals are available.

## market categories covered (v1)
- economic data releases: CPI, nonfarm payrolls, FOMC rate decisions — start here, best reference data available
- expand to political markets once economic workflow is stable
- expand to weather markets last — reference probability matching is harder

## time-to-resolution weighting
mispricing on a market resolving in 24 hours is more actionable than one resolving in 30 days. weight confidence score by proximity to resolution.

## what we are not scoring yet
- sports markets (covered by sports-analytics niche instead)
- low-liquidity or niche event markets
- intraday price movement patterns
