# decision 0001: football and basketball first

**date:** 2026-04-10
**status:** accepted
**applies to:** sports-v1 scope and data source selection

---

## decision

the first slice of sports-v1 covers NFL and NBA only. no other sports are in scope until at least one paying subscriber is active and the product is validated end-to-end for both sports.

**update 2026-04-10:** MLB was added to the current operational dev slice as a basic AI analysis path alongside NFL and NBA. the full data-integrated pipeline criteria in this note still apply before MLB receives production data source integration (odds api, injury data, etc.). the core reasoning about NFL and NBA sharing a pipeline applies equally to MLB in this slice.

---

## why NFL and NBA were chosen

**NFL:**
- weekly cadence keeps game volume manageable during build and validation — at most 16 games on a single Sunday
- the most liquid sports betting market in the US means signal data (sharp/public split, line movement) is the most reliable and well-sourced
- injury report timing is regulated and predictable — NFL publishes injury designations on a weekly schedule
- outdoor venues make weather a meaningful signal, adding a differentiator that other sports cannot fully replicate
- recreational bettors are most familiar with the format, which reduces the explanation burden during acquisition

**NBA:**
- nightly game volume (up to 13 games per night) keeps the product active outside football season
- back-to-back game scheduling creates a strong, measurable situational signal not present in other sports
- line movement and sharp/public split data are well-supported on the same data sources used for NFL
- covering NBA converts football-season users into year-round subscribers, which is the starter-to-pro upgrade path
- injury and availability data (rotowire) is available on the same plan tier as NFL

together, NFL and NBA share the same four data sources: the odds api, actionnetwork, rotowire, and open-meteo (NFL only). this means the collector, evaluator, and delivery pipeline can be built once and parameterized by sport, rather than rebuilt per sport.

---

## why soccer is deferred

**structural differences that require new work:**
- match timing varies significantly across leagues (MLS, EPL, Champions League) and time zones — the trigger system would need to handle overlapping windows and multi-league calendars
- the sharp/public split market for soccer is smaller and less standardized in the US; actionnetwork coverage is thinner
- injury and availability reporting has no equivalent to the NFL's structured injury designation system or rotowire's football/basketball feeds — sources vary by league
- weather is a factor for outdoor matches, but stadium data is international, which expands the lat/lon lookup scope significantly

**product and acquisition risk:**
- the target customer for v1 is a US recreational bettor who bets NFL and/or NBA; soccer adds a different acquisition profile with different trust signals
- validating signal quality for soccer requires a separate test set and evaluation pass
- adding soccer before v1 is validated would split engineering attention and delay the first paying subscriber

---

## what would need to be true before adding more sports

1. at least one active paying stripe subscription exists for sports-v1
2. the NFL and NBA pipelines are stable across a full season week (NFL) and a full game night (NBA) without manual intervention
3. a confirmed data source exists for sharp/public split data for the target sport
4. injury and availability data is available at a freshness level comparable to rotowire's NFL/NBA feeds
5. the trigger system has been validated for the new sport's scheduling pattern
6. a brief from the new sport has been evaluated manually against known outcomes for at least 10 games

these criteria apply to any sport — soccer, MLB, NHL, or college sports.

---

## how this supports disciplined scope and better engineering

**one shared pipeline, two parameterizations:**
having both NFL and NBA in v1 forces the platform to be parameterized correctly from the start. sport-specific behavior (weather null for NBA, back-to-back for NBA, trigger timing differences) must be handled via config and scoring rules rather than hard-coded per sport. this makes adding future sports a config extension, not a code rewrite.

**clear exit criteria:**
defining two sports up front means the definition of done is concrete: briefs produce correctly for a test NFL week and a test NBA week. without this constraint, scope would be ambiguous and validation harder to complete.

**prevents premature generalization:**
by limiting v1 to two well-understood sports with shared data sources, the team avoids building abstract multi-league infrastructure before the core pipeline is proven. the right abstraction will emerge from two real cases, not from speculation about a third.

**validates the signal model more thoroughly:**
NFL and NBA have meaningfully different signal profiles — NFL emphasizes weather and weekly situational context; NBA emphasizes back-to-back fatigue and nightly line movement. building and validating both in v1 stress-tests the evaluator and scoring model more thoroughly than a single sport would.
