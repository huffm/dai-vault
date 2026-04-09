# monetization: kalshi-analytics

## v1 model: subscription

### tiers

**free**
- daily brief covering top 5 flagged markets only
- next-day delivery (not same morning)
- goal: acquisition and trust-building

**starter — ~$29/month**
- daily brief covering all tracked markets
- same-morning delivery
- email delivery

**pro — ~$79/month**
- daily brief plus intraday alert when a watched market moves significantly
- custom watchlist: tenant can pin specific markets
- slack or webhook delivery
- weekly resolution accuracy summary

## what drives willingness to pay
- timeliness: same-morning delivery vs lagged
- alert layer: push on market price movement
- custom watchlist: coverage of markets the tenant actually trades

## future monetization options
- event packs: deep-coverage brief for high-stakes events (election night, fed decisions, jobs day)
- api tier for quant traders and forecasting researchers
- accuracy track record: publish resolution accuracy publicly as social proof and retention driver — this is a competitive moat if the scoring is honest

## stripe as truth
subscriptions managed through stripe. tenant access gated by active subscription. custom watchlist size enforced by tier.
