# signals: sports-analytics

## signal categories

### line movement
- opening line vs current line
- sharp vs public money indicators
- reverse line movement flags

### injury and availability
- official injury report status
- recent game logs for key players
- positional depth impact

### situational context
- home/away record, rest days, travel schedule
- divisional vs non-divisional games (NFL)
- back-to-back games and rest differential (NBA — one of the strongest situational signals)
- revenge game or schedule spot flags

### weather (NFL outdoor games only)
- wind speed and direction for passing/kicking games
- precipitation and temperature for run/pass tendency shifts
- not applicable to NBA — all arenas are indoors

### historical matchup
- head-to-head record at current spread range
- recent performance against similar opponents (same rest, similar travel)

## signal scoring (v1 approach)
each signal is scored +1 / 0 / -1 relative to the current line.
aggregate score flags whether the market looks efficient or skewed.
reverse line movement is the highest-weight single signal in v1 — if public money is heavy on one side but the line moves the other way, that is the sharpest flag we can surface.

## sport applicability summary
| signal | NFL | NBA |
|---|---|---|
| line movement | ✓ | ✓ |
| sharp vs public split | ✓ | ✓ |
| injury and availability | ✓ | ✓ |
| situational context | ✓ | ✓ (back-to-back is primary flag) |
| weather | ✓ outdoor venues only | — indoor arenas, skip |
| historical matchup | ✓ | ✓ |

## what we are not tracking yet
- player prop correlation models
- in-game live betting signals
- referee tendency data
- soccer, MLB, NHL signals — later expansion
