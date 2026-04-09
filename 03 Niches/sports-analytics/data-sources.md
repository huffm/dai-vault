# data-sources: sports-analytics

## primary sources

### the odds api
- endpoint: odds, lines, books, movement history
- covers: NFL, NBA, MLB, NHL, NCAAF, NCAAB
- cost: tiered by request volume
- use: core line data and sport schedule

### actionnetwork (scrape or api)
- endpoint: public betting percentages, sharp money indicators, line movement history
- cost: free scrape; paid api available
- use: sharp vs public split — most actionable signal after raw line data

### rotowire api
- endpoint: structured injury reports with position and depth impact
- cost: paid, reasonable for the data quality
- use: availability signals
- note: prefer over espn unofficial scraping — espn has ToS risk and no stable api contract

### open-meteo or weather.gov
- endpoint: hourly forecast by lat/lon
- cost: free
- use: weather signals for outdoor games

## secondary sources (v1 candidates)
- pro football reference / basketball reference: historical matchup and situational data
- fantasypros: secondary injury aggregation if rotowire is not yet licensed

## data freshness requirements
| signal | refresh frequency |
|---|---|
| lines | every 15 minutes during window |
| injury reports | daily, plus on-demand before game |
| sharp/public split | every 15 minutes during window |
| weather | 6 hours before game, once |
| historical | static per season |

## access pattern
all data pulled by fetcher agent at workflow trigger time. no streaming or websocket in v1. results cached per game until brief is dispatched.
