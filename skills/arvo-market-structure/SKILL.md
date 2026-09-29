---
name: arvo-market-structure
description: Read what a market is doing — trend, range, volatility, regime, session, gaps, timeframes — in Arvo's definitions, from Arvo's bars or a live feed. Use when asked what an instrument or the market is doing, to describe a chart, to classify a regime, to explain why a rule traded or failed in some period, or to turn a chart observation into a testable hypothesis. Not for advice about what to buy.
---

# Reading the market, the way Arvo does

Arvo describes markets; it never recommends. The output of this skill is a
description in Arvo's vocabulary and, if the person wants to act, a
hypothesis a study can test — never "buy" or "sell".

## Where the bars come from

| Need | Tool | Note |
|---|---|---|
| What a study saw | `query_market_data` (arvo) | the library on disk; ≤2000 bars from the end; fetches nothing |
| Regime per bar | `inspect_regime` (arvo) | labelled after the fact over the closes |
| Fresh or intraday bars, extended hours | `get_equity_historicals` (robinhood) | `interpolated: true` bars carry no information; `bounds` defaults to regular hours |
| Indicators Arvo does not compute (ADX, Bollinger, VWAP, Donchian, supertrend…) | `get_equity_technical_indicators` (robinhood) | `interval` is required; use to *describe*, not to feed a study |
| Depth, quote, fundamentals, earnings dates | robinhood read tools | context for a description |

Nothing from the broker side feeds a study: a study runs on the library only,
and the library is filled from the window. The broker MCP also holds order
tools; a market-reading task never calls one.

## Trend and range: the regime

Arvo's regime is one number. Over the last **20** closes, *net movement /
total movement* (Kaufman's efficiency ratio): a straight line scores 1.0, a
round trip scores 0.0. Above **0.35** the market went somewhere — trending
up or down by the sign of the net move; below it, **ranging**. Three labels,
not five: volatility is not a regime here, because the question is whether a
directional rule earned its result directionally.

- "Ranging" is not "flat". A range can be violent.
- A regime shorter than **10** bars is a handful, not a regime.
- Two regimes disagreeing by ≥ **5 points** of excess return is a caveat on
  the verdict, never a trading rule.

**The label is hindsight.** It is computed over the completed window and is
legitimate for describing a result, illegitimate for taking one. "This rule
made its money in the trending third and gave it back in the range" is
honest. "So trade it in trends" needs a real-time detector, which Arvo does
not have; a backtest filtered on this label is look-ahead of the most
flattering kind. A rule may filter only on a series computed from what was
known at that bar (SMA slope, ATR level, a channel) — see `arvo-indicators`.

## Volatility

ATR (`atr_period` 14 by default) is the unit Arvo thinks in: stops are a
multiple of it, position size is risk divided by it. Describe volatility as
ATR as a share of the close, and as where today's ATR sits against its own
recent history. A move of ten times the median true range is what the data
checker flags as an outlier — and also what a crash or an earnings gap looks
like; say which it was.

Annualise by the interval's own periods per year, never a hardcoded 252:
on five-minute bars that understates Sharpe nine times over.

## Sessions and gaps

Regular hours are **09:30–16:00 New York**. Extended-hours bars are kept out
of the library at fetch and flagged if on disk, because an opening range or
a session VWAP built on a 04:00 print is the thinnest print of the day. No
early-close calendar: the day after Thanksgiving counts to 16:00.

An intraday rule can hold overnight, and the close-to-open step is a
different exposure — news, earnings, the overnight drift — that no intraday
signal chose. Findings split the curve at session boundaries and report how
much was earned while the market was closed. Read that before reading an
intraday verdict as a statement about entries.

A gap on a daily chart: open vs prior close, in ATRs. A gap of 30%+ that
lands within 1% of a whole-number ratio is a split the vendor missed, not a
move.

## Timeframes

The library holds daily (`data/`), five-minute and one-minute series. The
interval is part of the experiment: the same rule at 1day and 5minute is a
different finding. Describe the higher timeframe first (regime, ATR, where
price sits against its 20-bar channel), then the working one. Do not stack
signals across timeframes into a confidence score; Arvo has no such number
and neither should you.

## Volume, depth, flow

Volume is a bar field a rule can read (`volume` against an SMA of it). Depth
(`get_equity_price_book`, L2, four symbols at most) and quotes describe the
moment, not the series; imbalance in the top levels leans a few seconds of
direction and nothing a daily rule can use.

## From observation to hypothesis

A chart observation becomes useful only as a premise: one line, before the
logic, stated as what would have to be true. "AAPL pulls back to its 20-day
average in uptrends and resumes" is a premise; a rule sketch is
`arvo-indicators`; the test is `arvo-research`. Chart patterns that are not
indicator conditions — head and shoulders, triangles, order blocks — cannot
be tested here and should be said to be untestable rather than approximated
silently. The literature note the SKE-Labs pattern skills carry applies:
candlestick rules were not generally profitable on large US stocks.

## Do not

- Say what the market "will" do, or what to buy. Describe, then propose a test.
- Read a hindsight regime label as a filter.
- Mix extended-hours bars into a regular-hours description without saying so.
- Describe one bar as a trend, or ten as a regime.
