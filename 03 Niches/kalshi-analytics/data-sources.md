# data-sources: kalshi-analytics

## primary sources

### kalshi api (official)
- endpoint: active markets, prices, volume, order book, market history
- cost: free with api key
- use: core market data for all workflows

### polymarket api or subgraph
- endpoint: equivalent market prices on polymarket
- cost: free
- use: external reference probability for divergence scoring

### metaculus api
- endpoint: community probability estimates for matching questions
- cost: free
- use: forecasting aggregator reference for economic and political markets

### open-meteo or weather.gov
- endpoint: probabilistic weather forecast
- cost: free
- use: reference probability for kalshi weather markets

## secondary sources (v1 candidates — promote these to primary for economic markets)
- cme fedwatch (free): model probability for fed rate decision markets — most authoritative single reference for FOMC contracts
- econoday or briefing.com (free): economic calendar with consensus estimates for CPI, NFP, and other releases
- election forecasting aggregators (polymarket aggregate, 538-equivalent): political market reference

## data freshness requirements
| signal | refresh frequency |
|---|---|
| kalshi market prices | daily at brief trigger |
| kalshi price movement | daily (or per alert trigger) |
| external reference probabilities | daily |
| scheduled event calendar | daily |

## access pattern
all data pulled by fetcher agent at daily workflow trigger and at event-alert trigger. no persistent streaming connection in v1. results cached per run and discarded after brief is dispatched.
