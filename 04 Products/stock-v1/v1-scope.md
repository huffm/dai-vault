# v1 scope: stock-v1

## in scope

### earnings brief (primary workflow)
- triggers 48 hours before each earnings event on the tenant's watchlist
- covers per stock:
  - upcoming earnings date and options-implied expected move
  - analyst estimate revision trend (past 30 days)
  - earnings surprise streak (beat/miss/in-line)
  - price vs 50d and 200d moving averages
  - volume trend on recent up vs down days
  - recent insider buys or sells (SEC Form 4, past 30 days)
  - short interest and days-to-cover
  - sector relative strength vs S&P 500
  - aggregate signal score: constructive / mixed / deteriorating
  - 3-5 sentence narrative

### weekly watchlist brief (secondary workflow)
- runs Friday after market close, delivered Sunday evening
- one-page summary of full watchlist with flags for any signal shift since last brief

### delivery
- email (starter), slack/webhook (pro)

### tiers
- free: 3 stocks, weekly brief only
- starter: 10 stocks, earnings briefs + weekly brief
- pro: 50 stocks, earnings + weekly + signal-shift alert + monthly accuracy summary

## out of scope for v1
- options flow or unusual activity scanner
- conference call transcript sentiment (v1 candidate post-launch)
- intraday alerts beyond signal-shift
- portfolio position tracking or P&L

## key decisions for v1 build
- data sources: Polygon.io (price/technicals), SEC EDGAR (Form 4 filings), Econoday or Zacks (earnings calendar/consensus), Alpha Vantage or FinancialModelingPrep (fundamentals/revisions), Finviz (short interest), FRED (macro)
- 13F institutional ownership: treat as background context only — 45-day lag makes it unsuitable as a live signal
- trigger: earnings calendar scan runs daily; brief fires when event is 48 hours out

## definition of done for v1
- earnings brief produces correctly for a 5-stock test watchlist across one full earnings cycle
- weekly brief generates and delivers on schedule
- at least one newsletter operator or RIA using it before expanding watchlist cap
