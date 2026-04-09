# signals: crypto-analytics

## signal categories (ordered by v1 priority)

### derivatives market
- funding rates: positive = longs paying, negative = shorts paying
- open interest trend: rising OI with price = confirmation; divergence = warning
- liquidation clusters: where forced selling or buying pressure concentrates
- note: derivatives signals move fastest and are most actionable for short-term traders

### on-chain activity
- exchange inflows and outflows (accumulation vs distribution pressure)
- active addresses trend
- large wallet movement (whale activity flags)

### market structure
- price relative to key moving averages (20d, 200d)
- recent higher highs / lower lows trend
- volume profile relative to recent average

### sentiment
- fear and greed index (primary sentiment signal in v1 — free and reliable)
- major news or regulatory event in the past 48 hours
- social volume spike: secondary, added when LunarCrush is integrated

## signal scoring (v1 approach)
each signal category scored bullish / neutral / bearish.
aggregate confluence score flags high-conviction vs mixed setup.

## what we are not tracking yet
- defi protocol-specific signals
- cross-chain bridge flows
- nft market activity
