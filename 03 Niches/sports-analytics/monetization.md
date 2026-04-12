# monetization: sports-analytics

## v1 model: subscription

### tiers

**free**
- NFL only, one game brief per week (best game of the week by signal score)
- same-day delivery, no alert layer
- goal: show the product working with the highest-trust sport first

**starter — ~$29/month**
- NFL all games in season + NBA all games in season
- brief delivered morning of game day (NFL) or morning of game day (NBA)
- email delivery

**pro — ~$79/month**
- NFL and NBA, all games
- brief delivered 24h before kickoff or tip-off (more time to act)
- re-alert if line moves 2+ points after initial brief
- slack or webhook delivery

## what drives willingness to pay
- timeliness: earlier delivery = more time to act
- sport breadth: NFL free, NBA requires paid — year-round coverage is a clear upgrade hook
- alert layer: async push when a game looks sharp

## future monetization options
- one-time report packs (e.g. playoffs bundle, finals preview)
- api access for syndicates
- white-label for sports media outlets
- soccer, MLB, NHL expansion — later tiers or add-on pricing

## stripe as truth
subscriptions managed through stripe. tenant access tied to active subscription status. no manual overrides in production.
