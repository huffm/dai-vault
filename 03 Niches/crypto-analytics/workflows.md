# workflows: crypto-analytics

## v1 core workflow: daily market brief

### trigger
runs at 06:00 UTC — before European session open (08:00 CET). gives traders in EU and early US time zones the setup before they make position decisions. configurable per tenant timezone.

### steps
1. fetch price and volume data for watchlist assets
2. pull on-chain metrics (exchange flows, active addresses)
3. pull derivatives data (funding rates, open interest, liquidation levels)
4. pull sentiment indicators (fear/greed, social volume)
5. score each signal category per asset
6. run summarizer agent to produce structured brief per asset
7. aggregate into a single daily brief for the tenant's watchlist
8. deliver to tenant via email, webhook, or dashboard

## brief output format
- date and market snapshot header
- per-asset section: signal table + 3-5 sentence narrative
- overall market tone: risk-on / neutral / risk-off
- notable flags: any signal with extreme reading called out

## agent roles used
- collector: pulls raw data per source
- evaluator: scores each signal category per asset
- synthesizer: writes narrative from scored signals
- compliance: confirms output does not constitute regulated financial advice or violate scope constraints
- delivery: formats and dispatches the brief to the tenant's configured channel

## funding rate alert (v1 stretch)
if funding rate crosses an extreme threshold (above +0.1% or below -0.05% for 8h rate), trigger an out-of-schedule alert regardless of daily cadence. this is the single most useful intraday alert we can build cheaply.

## what this workflow does not do
- does not execute trades
- does not track position-level performance
- does not run intraday on a full schedule (v1 is daily + funding extreme alerts only)
