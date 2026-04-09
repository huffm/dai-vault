# niche comparison

**date:** 2026-04-09
**scope:** sports-analytics, crypto-analytics, stock-analytics, kalshi-analytics
**purpose:** rank the four niches across six dimensions to inform where to cut the first wedge

---

## scoring table

| dimension | sports | crypto | stock | kalshi |
|---|---|---|---|---|
| clarity of customer pain | ★★★★★ | ★★★★☆ | ★★★★☆ | ★★★☆☆ |
| speed to v1 | ★★★★★ | ★★★★☆ | ★★★☆☆ | ★★★★☆ |
| monetization potential | ★★★★☆ | ★★★☆☆ | ★★★★★ | ★★☆☆☆ |
| data complexity | ★★★★☆ | ★★★☆☆ | ★★☆☆☆ | ★★★★★ |
| platform fit | ★★★★★ | ★★★★★ | ★★★★☆ | ★★★★★ |
| founder fit (assumed) | ★★★☆☆ | ★★★☆☆ | ★★★★☆ | ★★★☆☆ |

*data complexity: more stars = simpler. fewer stars = harder/more sources.*

---

## per-niche notes

### sports-analytics
**strongest on:** pain clarity, speed to v1, platform fit

the customer pain is the most time-bound and specific of the four. a bettor has a window. lines move. they either have context before it closes or they don't. that sharpness makes acquisition easier — the value is felt immediately on first use.

the competition is touts, which people actively distrust. walking away from a tout and toward a transparent, signal-based brief is a story people are ready to hear.

NFL-only v1 is genuinely small scope: 16 games/week, one brief type, three or four data sources, one delivery window. fastest path to a working product.

**risks:** seasonal (NFL is September–January). off-season retention requires expanding to NBA or other sports, which adds scope. not a fit if there is no domain interest in sports betting.

---

### crypto-analytics
**strongest on:** platform fit, speed to v1

the pain is real. on-chain data tools are good but synthesis is missing. the daily morning brief fits the platform workflow model perfectly.

the challenge is competitive density. Glassnode, CoinGlass, LunarCrush, and dozens of telegram signal channels already exist. the customer knows the data exists — they just don't want to check five dashboards. that's a real gap but it's a convenience gap, not a tool gap. convenience gaps are harder to monetize at $29–$79/month because the alternative is "just check it yourself."

crypto traders also churn aggressively and have low trust in paid signal services after years of scam channels. first-paid conversion will be slower than sports.

**risks:** crowded space, churn-prone audience, glassnode free tier limits early signal quality.

---

### stock-analytics
**strongest on:** monetization potential, founder fit

the largest long-term addressable market. newsletter operators and small RIAs are business customers with real budgets — they have a publishing obligation, a professional reputation at stake, and a direct reason to pay for context. $149/month is credible to them in a way it isn't to a retail bettor.

the pre-earnings brief is a strong, defensible wedge. the customer pain is concrete and the delivery moment (48 hours before earnings) is clear.

slower to v1 than sports or kalshi because data complexity is highest: Polygon, SEC EDGAR, earnings calendar, analyst revisions, options expected move. more integration work before the first brief works end-to-end.

**risks:** takes longer to build. first-paid customer may require a more deliberate B2B sales motion (outreach to newsletter operators) rather than community-driven acquisition.

---

### kalshi-analytics
**strongest on:** data simplicity, platform fit

the most technically simple niche — kalshi API plus a handful of free reference sources (CMEFedWatch, Econoday) covers the full signal picture for economic data markets. the workflow is clean: fetch, compare to reference, score, brief.

the problem is TAM. kalshi's US active trader base is still small. the economic data market traders who would pay are a subset of a subset. this niche may never support the revenue of sports or stock at v1 scale.

best framing: kalshi-analytics is an experiment with a 60-day test — if 50 paying users don't materialize, it may not be a standalone business yet. watch for platform growth as a re-entry signal.

**risks:** small addressable market today. may be better as a feature of another niche product than a standalone subscription.

---

## comparison summary

| niche | best case | main risk |
|---|---|---|
| sports | fastest to first paying user | seasonal, requires domain interest |
| crypto | clean platform fit | crowded, convenience not tool gap |
| stock | highest revenue ceiling | slower to build, needs B2B motion |
| kalshi | simplest data layer | TAM too small right now |

---

## recommended first wedge: sports-analytics (NFL)

**why:**

the customer pain is the most specific and time-bound. the competition (touts) is something customers actively want to leave. the v1 scope is genuinely small — one sport, one brief type, one delivery window per week. data sources are affordable and well-documented.

most importantly: you can launch something useful, get real users in front of it, and get a clear signal on whether it works — all within a few weeks. if it resonates, expanding to NBA and adding the line-movement re-alert gives you a natural upgrade path. if it doesn't, you've learned fast.

**stock-analytics is the better long-term business**, but it requires more upfront build time and a different customer acquisition motion. start sports, validate the platform workflow against a tight use case, then layer stock-analytics on the same infrastructure.

kalshi-analytics is worth running as a low-cost parallel experiment if there is existing interest — the build is lightweight and the workflow is the same pattern. do not let it distract from the primary wedge.
