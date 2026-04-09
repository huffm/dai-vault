# thesis: kalshi-analytics

## core problem
kalshi traders know what markets exist but have no fast way to assess which contracts are mispriced before a scheduled event resolves them. they check the price, check the news, and make a guess. the reference probabilities — CME fed futures, economic consensus, model forecasts — are public but scattered. no one assembles them into a brief before the window closes.

## what we do
24 hours before a major scheduled economic event (CPI, NFP, FOMC), produce a brief that compares the kalshi market price to every available reference probability. one clear output: does this contract look mispriced relative to what is knowable right now?

## the wedge
economic data release markets only, to start. CPI, nonfarm payrolls, and FOMC rate decisions are the most liquid kalshi markets and have the best external reference data. nail these before expanding to political or weather markets.

## why now
kalshi's api is public and well-documented. CME fedwatch, BLS consensus estimates, and economic calendars are free. the synthesis gap is real and no one has closed it for prediction market traders specifically.

## tam note
kalshi's US active trader base is relatively small today. treat this niche as an experiment with a clear signal: if you can't find 50 paying users within 60 days of launch, the TAM may not be there yet. watch for platform growth as a trigger to scale.

## what we are not
- we are not a trading bot or automated execution layer
- we do not guarantee resolution outcomes
- we do not provide regulated financial advice
- we do not cover polymarket or other prediction platforms in v1
