# signals: stock-analytics

## signal categories

### earnings and fundamentals
- upcoming earnings date and expected move (options-implied)
- revenue and EPS trend over last 4 quarters
- analyst estimate revisions in past 30 days
- earnings surprise history (beat / miss / in-line streak)

### technical structure
- price relative to 50d and 200d moving averages
- recent trend: higher highs / lower lows
- volume on recent up days vs down days
- proximity to key support or resistance levels

### insider and institutional activity
- recent insider purchases or sales (sec form 4) — most actionable, data is fresh
- short interest and days-to-cover ratio
- institutional ownership change (13F filings) — note: 45-day lag makes this context, not signal. flag as background only.

### macro backdrop
- sector relative strength vs sp500
- rate sensitivity flag (high duration stocks vs value)
- recent macro events affecting the sector

## signal scoring (v1 approach)
each signal category scored favorable / neutral / unfavorable relative to the current setup.
aggregate score flags whether the setup looks constructive or deteriorating.

## what we are not tracking yet
- options flow (unusual activity scanner) — v2
- supply chain or credit risk signals — v2
- conference call transcript sentiment — v1 candidate: feasible with summarizer agent on earnings call transcripts, worth testing once brief workflow is stable
