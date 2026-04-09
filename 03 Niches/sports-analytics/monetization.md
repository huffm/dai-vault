# monetization: sports-analytics

## v1 model: subscription

### tiers

**free**
- NFL only, one game brief per week (best game of the week by signal score)
- same-day delivery, no alert layer
- goal: show the product working, not a degraded version of it

**starter — ~$29/month**
- NFL all games in season
- brief delivered morning of game day
- email delivery

**pro — ~$79/month**
- all supported sports
- brief delivered 24h before kickoff (more time to act)
- re-alert if line moves 2+ points after initial brief
- slack or webhook delivery

## what drives willingness to pay
- timeliness: earlier delivery = more time to act
- sport breadth: more sports = more value per week
- alert layer: async push when a game looks sharp

## future monetization options
- one-time report packs (e.g. playoffs bundle)
- api access for syndicates
- white-label for sports media outlets

## stripe as truth
subscriptions managed through stripe. tenant access tied to active subscription status. no manual overrides in production.
