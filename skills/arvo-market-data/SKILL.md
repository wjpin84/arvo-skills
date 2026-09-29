---
name: arvo-market-data
description: Understand the bars a study runs on — the library layout, venues, intervals, split vs total-return adjustment, the data-quality checks, comparing two vendors, staleness, and when to use the broker feed instead. Use when asked why a result looks wrong, whether data is missing or bad, what an instrument id means, why two sources disagree, why a finding went stale, or what a bar file contains.
---

# The data under a finding

Bars are fetched to files, pinned by content hash, and never read live
during a run. The same study over the same library gives the same answer
next year, and a finding records which library version produced it.

## The library

```
<project>/data/<SYMBOL>.<VENUE>.csv      daily: date,open,high,low,close,volume
<project>/data/5minute/, data/1minute/    intraday series
<project>/data/dividends/                 distributions, where a source serves them
<project>/staleness.json                  findings whose data changed underneath them
```

An instrument is `SYMBOL.VENUE`. The venue names **where the bars came
from**, not where an order goes: `YF` (Yahoo), `RH` (Robinhood), `AIEX`,
`ACRYPTO`, `SIM` (synthetic). `AAPL.RH` and `AAPL.YF` are two series of one
stock and may disagree. `list_instruments` says what is held, at which
interval, over what range. Filling it is the window's job (or the
`universes` job, every six hours); nothing an agent calls contacts a vendor.

A bar carries a timestamp, not a date; the interval (seconds to weeks) is
part of the experiment and of annualisation.

## Adjustment

Every source ships **split-adjusted** prices, which is right — raw prices
make a split look like a crash. Split-adjusted is not **total return**:
dividends are absent and nothing credits them as cash, so the benchmark,
which holds through every ex-date, forgoes more than a rule in the market
40% of the time. The reported excess return is therefore overstated in the
rule's favour on every dividend payer. Arvo measures this **dividend gap**
beside the result rather than folding it in, and refuses to subtract it on a
total-return dataset where it would be double counting. Read it beside the
margin over buy-and-hold.

The broker feed has the same choice: `adjustment_type` `split` (default),
`none`, or `all`.

## What the checker looks for

Parsing refuses **impossible** bars — high below low, a negative close, a
malformed date. The quality checks are for bars that are individually
possible and collectively wrong. Nothing here stops a run; the observations
are attached to the result, because whether a 43% single-day fall is a data
fault or March 2020 is a judgement the checker cannot make.

| Check | Threshold | Severity |
|---|---|---|
| Two bars at one instant | any | fault |
| Stalled feed | 3 consecutive bars with identical O, H, L and C | fault |
| Missing trading week | > 5 calendar days between daily bars (1 day, or 1 hour quiet, on a market that never closes) | fault |
| Outlier | true range > 10× the median | suspect — a crash looks the same |
| Unadjusted split | a move ≥ 30% within 1% of a whole-number ratio | suspect |

The bar for adding a check: one that fires on legitimate data is worse than
none, because it teaches people to skip the warnings.

## Two vendors

A series checked against itself catches only the impossible. A close off by
forty cents is well formed. The only independent version of a price is
somebody else's, so the window can compare sources (`CompareSources`). Two
vendors disagreeing is normal, and the disagreement is **classified before
it is counted**:

- **Adjustment** — every bar differs by one factor: not comparable until one is restated.
- **Session** — extended hours in or out changes which bars exist.
- **Volume basis** — consolidated tape vs primary exchange differ 3× on prices that agree.
- Then the residual: prices more than **10 bps** apart, counted.

"4,812 bars disagree" across any of the first three is true and useless.

## Stale

A finding is stale when its data was refetched and differs, or its ruleset
changed. `rank_findings` rows say so; `staleness.json` lists them. A stale
finding is history, not evidence: re-run it.

## The broker feed is not the library

`get_equity_historicals` (robinhood) gives fresh bars at fixed intervals,
regular hours by default, split-adjusted by default, and marks bars it
synthesised with `interpolated: true`. Use it to look at now. It does not
reach a study, and a description built on it should say so.

## Do not

- Explain a result before checking the finding's data-quality notes.
- Compare `X.RH` and `X.YF` numbers without asking which adjustment each has.
- Treat an outlier flag as an error, or a split flag as a move.
- Read a total-return margin and a split-adjusted one as the same number.
