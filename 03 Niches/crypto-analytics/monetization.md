# monetization: crypto-analytics

## v1 model: subscription

### tiers

**free**
- BTC only, current daily brief (no lag)
- no alert layer, no alt coverage
- goal: show the product working — a lagged brief teaches nothing about whether it's useful

**starter — ~$29/month**
- up to 5 assets on watchlist
- daily brief delivered same morning
- email delivery

**pro — ~$79/month**
- unlimited watchlist assets
- daily brief plus intraday alert when a signal score changes materially
- slack or webhook delivery
- weekly summary report

## what drives willingness to pay
- watchlist size: more assets = more daily coverage
- timeliness: same-day vs lagged delivery
- alert layer: push notification on signal regime change

## future monetization options
- one-time event reports (halving, major macro events)
- api tier for funds and developers
- white-label for crypto newsletters and influencers

## stripe as truth
subscriptions managed through stripe. tenant access tied to active subscription status. watchlist size enforced by tier.
