# v1 scope: crypto-v1

## in scope
- BTC and ETH (free tier: BTC only)
- daily morning brief per asset, covering:
  - funding rate and trend
  - open interest direction
  - liquidation cluster summary
  - exchange inflow/outflow signal
  - active addresses trend
  - price vs key moving averages
  - fear and greed index
  - aggregate tone: risk-on / neutral / risk-off
  - 3-5 sentence narrative
- email delivery for starter tier
- slack/webhook delivery for pro tier
- funding rate extreme alert (above +0.1% or below -0.05% for 8h rate) — pro only
- watchlist expansion up to 5 assets (starter), unlimited (pro)

## out of scope for v1
- DeFi protocol signals
- NFT market activity
- cross-chain bridge flows
- social volume (LunarCrush) — secondary, added post-v1
- intraday full schedule beyond funding alert

## key decisions for v1 build
- data sources: CoinGecko (price/volume), CoinGlass (funding/OI/liquidations), Alternative.me (fear/greed)
- add Glassnode paid once monthly revenue justifies it; start with CoinGlass on-chain proxies
- trigger: 06:00 UTC daily + funding threshold breach
- brief format: fixed markdown template rendered to email and webhook

## definition of done for v1
- daily brief produces correctly for BTC and ETH on schedule
- funding alert fires correctly on threshold breach in test
- at least one paying subscriber before expanding watchlist feature
