# data-sources: crypto-analytics

## primary sources

### coingecko api
- endpoint: price, volume, market cap, historical data
- cost: free tier available; paid for higher limits
- use: price and volume data for watchlist assets

### glassnode
- endpoint: on-chain metrics (exchange flows, active addresses, whale activity)
- cost: paid, tiered by metric depth
- use: core on-chain signal layer
- note: free tier is very limited — start with CoinGlass for derivatives and add Glassnode paid once revenue justifies it. alternatively, use CryptoQuant as a lower-cost on-chain alternative.

### coinglass
- endpoint: funding rates, open interest, liquidation heatmaps
- cost: free tier available; api requires paid plan
- use: derivatives signal layer

### alternative.me fear and greed index
- endpoint: daily fear and greed score
- cost: free
- use: sentiment signal

## secondary sources (v1 candidates)
- nansen: labeled wallet activity and smart money flows
- lunarcrush: social volume and engagement metrics
- the block data: institutional and macro context

## data freshness requirements
| signal | refresh frequency |
|---|---|
| price and volume | daily at brief trigger |
| on-chain metrics | daily |
| funding rates | daily (or per alert trigger) |
| fear and greed | daily |
| liquidation clusters | daily |

## access pattern
all data pulled by fetcher agent at daily workflow trigger. no streaming in v1. results cached per run and discarded after brief is dispatched.
