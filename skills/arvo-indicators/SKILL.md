---
name: arvo-indicators
description: What each indicator Arvo computes means and how the engine evaluates it, plus the common technical setups — crossovers, pullbacks, RSI reversion, MACD, trend and volume filters, breakouts — as Arvo rule sketches or an honest "not expressible here". Use when asked to turn a trading idea, chart signal or TradingView strategy into an Arvo rule, to explain what a rule's condition means, why a rule never fires or fires too often, or which compiled rule already covers an idea.
---

# Indicators, as the engine evaluates them

The full rule syntax is in `arvo-rules`. This is what the pieces *mean*
and how the engine reads them, so a sketch here runs the way it reads.

## How evaluation works

- Every indicator returns nothing until its window is full: a ten-bar
  average over three bars is not a ten-bar average, and treating it as one
  manufactures signal at the start of every backtest. A condition that reads
  an unknown value neither fires nor remembers. MACD is warm at
  `slow + signal − 1` bars; RSI at `period + 1`.
- `cross_above [a, b]` fires on the bar `a > b` first becomes true after
  being false; `cross_below` the reverse. Once per crossing. The crossing's
  memory is **forgotten after a stop, level exit or halt**, so re-entry
  needs a fresh cross, not a standing relation.
- `MAX` and `MIN` are over the last `period` bars **including the current
  one**. `close > MAX(high, 20)` can never be true.
- No `>=`, no arithmetic, no lag operator, no second instrument, no shorts.
- The journal records the first comparison's `a − b` as the signal's
  strength; put the comparison you want to read about first.
- `fixed: { "trade_size": N }` in a ruleset is the quantity the rule asks
  for before the gate sizes it; it is not a rule parameter.

## What each indicator is for

| kind | reads | use it for | watch |
|---|---|---|---|
| `SMA` | any field | trend level, a slow baseline, a volume baseline | lags by half its period |
| `EMA` | any field | a faster baseline; pullback anchor | seeded with an SMA, so warm-up equals the period |
| `ATR` | H, L, C | volatility in price units; the unit stops and sizing use | price-scaled: `atr > 3` means different things on a $10 and a $500 stock, and there is no division |
| `RSI` | any field | overbought/oversold thresholds; momentum extremes | Wilder smoothing; on trending names it pins at 70+ for weeks |
| `MACD` | any field | momentum turn (line vs signal), acceleration (histogram vs 0) | one declaration is one series — declare `line: macd` and `line: signal` separately |
| `MAX`/`MIN` | default H / L | the channel price is *inside*; range width | includes the current bar, so it is a level to sit above or below, never a level to break |

## Setups you can write

Sketches, not recommendations; each is a premise to test with `run_study`.

**Trend filter + pullback entry** — the shape `regime_pullback` in the project already uses:
```json
"entry": { "and": [ { ">": [{"var":"close"},{"var":"trend"}] },
                    { "cross_above": [{"var":"close"},{"var":"fast"}] } ] },
"exit":  { "or":  [ { "cross_below": [{"var":"close"},{"var":"fast"}] },
                    { "<": [{"var":"close"},{"var":"trend"}] } ] }
```
with `fast: EMA 20`, `trend: SMA 200`. The `>` is causal — it reads what was
known at the bar — which is the only kind of regime filter a rule may use.

**Moving-average crossover** — `twin_cross`; the control, not an edge.

**RSI reversion** — `rsi: RSI 14`; entry `{"<": [{"var":"rsi"}, 30]}`,
exit `{">": [{"var":"rsi"}, 70]}`. Fires every bar the condition holds, so
it re-enters the bar after a stop-out; wrap the entry in a `cross_below`
against the literal to fire once.

**MACD turn** — `line: MACD(12,26,9, line: macd)`, `sig: … line: signal`;
entry `cross_above [line, sig]`. Histogram flip: `hist: … line: histogram`,
entry `cross_above [hist, 0]`.

**Volume confirmation** — `vol20: SMA(input: volume, 20)`; add
`{">": [{"var":"volume"},{"var":"vol20"}]}` to an `and`.

**Position in the channel** — `upper: MAX(high, 20)`, `lower: MIN(low, 20)`;
`close` is always between them, so use them only against *each other* or
against a moving average (`{">": [{"var":"fast"},{"var":"lower"}]}`).

## Ideas that are not expressible as data

Say so, and name the compiled rule if one covers it (`list_strategies`):

| idea | why not | compiled |
|---|---|---|
| Donchian / range breakout | needs yesterday's channel; MAX includes today | `momentum_breakout`, `volatility_breakout` |
| Opening-range break, session VWAP stretch | session-anchored; refuse daily bars | `opening_range`, `vwap_reversion` |
| ATR as a fraction of price, Bollinger, Keltner, z-score | need division or a deviation | — |
| Stochastic, ADX, supertrend, Ichimoku, VWAP as a series | not in the indicator set | describe with `get_equity_technical_indicators` (robinhood); cannot feed a study |
| Shorts, pairs, spreads, another symbol's series | long only, one instrument | `cross_sectional_momentum` ranks a book; `put_spread` and the `zero_dte_*` rules trade options off an underlying |
| Chart patterns, order blocks, fair-value gaps | not indicator conditions | — |

`buy_and_hold` is the benchmark every study is scored against, not a rule
to pick.

## Reading a rule that misbehaves

- Never fires: a comparison against `MAX`/`MIN` of its own field; a `>=`
  written as `>` on integer-like data; an indicator whose warm-up exceeds
  the in-sample window.
- Fires every bar: a `>`/`<` entry where a cross was meant.
- Fires once and stops: a cross whose memory was reset by a stop and never
  re-crossed.
- Trades in one regime only: read the regime column in the finding's
  trades; it is a caveat on the verdict, not a filter to add
  (`arvo-market-structure`).

## Do not

- Add a hindsight regime label as a condition.
- Write a threshold in price units and call it a volatility filter.
- Approximate an untestable pattern with a nearby condition without saying
  the premise changed.
