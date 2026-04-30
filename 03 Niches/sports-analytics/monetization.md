# monetization: sports-analytics

## v1 model: subscription

### tiers

**free**
- NFL only, one game artifact per week (best game of the week by signal score)
- same-day delivery, no alert layer
- history includes the post-close audit note for that free read
- goal: show the product working with the highest-trust sport first

**starter — ~$29/month**
- NFL all games in season + NBA all games in season
- artifact delivered morning of game day (NFL) or morning of game day (NBA)
- email delivery
- full artifact archive and saved reads

**pro — ~$79/month**
- NFL and NBA, all games
- artifact delivered 24h before kickoff or tip-off (more time to act)
- re-alert if line moves 2+ points after initial artifact
- slack or webhook delivery
- priority history with line-move audit trail

## what drives willingness to pay
- timeliness: earlier delivery = more time to act
- sport breadth: NFL free, NBA requires paid — year-round coverage is a clear upgrade hook
- alert layer: async push when a game looks sharp
- trust and accountability: archived reads with post-close review make the product easier to judge
- simplicity: users pay to avoid research sprawl and get to clarity faster

## future monetization options
- one-time report packs (e.g. playoffs bundle, finals preview)
- white-label delivery for newsletters, discord communities, or media brands
- api access for syndicates
- soccer, MLB, NHL expansion — later tiers or add-on pricing

## stripe as truth
subscriptions managed through stripe. tenant access tied to active subscription status. no manual overrides in production.
