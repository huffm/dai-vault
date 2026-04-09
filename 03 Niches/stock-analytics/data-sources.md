# data-sources: stock-analytics

## primary sources

### polygon.io
- endpoint: price, volume, ohlcv, technicals, options data
- cost: paid, tiered by data depth and history
- use: core price and technical signal layer

### sec edgar (public api)
- endpoint: form 4 (insider transactions), 13F (institutional holdings), 8-K (material events)
- cost: free
- use: insider and institutional activity signals

### earnings calendar: econoday or zacks earnings calendar
- endpoint: earnings dates, estimate consensus, expected move
- cost: free tier available; paid for bulk api access
- use: earnings event trigger and estimate consensus data
- note: earningswhispers does not offer a stable public api — do not plan on it for programmatic access

### alpha vantage or financialmodelingprep
- endpoint: income statement, analyst revisions, sector performance
- cost: free tier available; paid for higher limits
- use: fundamental and estimate revision signals

## secondary sources (v1 candidates)
- finviz: short interest, technical screener data
- fred (federal reserve economic data): free, authoritative macro backdrop signals
- yahoo finance (unofficial): secondary price fallback only — no stable api contract

## data freshness requirements
| signal | refresh frequency |
|---|---|
| price and volume | daily at brief trigger |
| earnings calendar | daily scan |
| analyst revisions | weekly |
| insider filings | daily (edgar rss feed) |
| sector performance | weekly |

## access pattern
all data pulled by fetcher agent at workflow trigger. no streaming in v1. results cached per brief run, discarded after dispatch.
