# monetization: stock-analytics

## v1 model: subscription

### tiers

**free**
- up to 3 stocks on watchlist
- weekly brief only, no earnings alerts
- goal: acquisition and trust-building

**starter — ~$29/month**
- up to 10 stocks on watchlist
- earnings briefs + weekly watchlist brief
- email delivery

**pro — ~$149/month**
- up to 50 stocks on watchlist (RIA / newsletter operator use case)
- earnings briefs + weekly brief + intraday alert on significant signal shift
- slack or webhook delivery
- monthly accuracy summary: how well did scored signals predict earnings direction

## what drives willingness to pay
- watchlist size: more coverage = more value per week
- earnings coverage: pre-earnings brief is a high-value moment
- alert layer: push when signal changes before a catalyst

## future monetization options
- sector packs (e.g. biotech earnings coverage, energy sector brief)
- api tier for RIAs and small funds
- white-label for financial newsletters

## stripe as truth
subscriptions managed through stripe. tenant access gated by active subscription. watchlist size enforced per tier.
