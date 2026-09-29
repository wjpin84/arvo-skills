---
name: arvo-risk-and-costs
description: How Arvo sizes, stops, refuses and costs a trade, and how to read a result through them. Use when asked about position sizing, stops, the risk model or risk.json, why the gate refused an order, drawdown or daily-loss halts, correlation caps, transaction costs and cost tiers, slippage, options stress, whether a live session still matches its finding, or what promotion to real money requires.
---

# Risk and costs in Arvo

One gate sizes or refuses every entry, in a backtest and live, with the
same code — so a finding is a statement about what would actually have
happened. **An exit never asks the gate.** A refusal is never silent; it is
on the record with a name.

## The risk model

The project's `risk.json` (`GetRiskModel` gives the path; the Risk tab
shows it; Arvo's defaults apply when there is none):

```json
{
  "stop_atr_multiple": 2.0,
  "atr_period": 14,
  "risk_per_trade": 0.01,
  "max_position_fraction": 1.0,
  "max_drawdown": null,
  "max_concurrent_positions": null,
  "max_daily_loss": null,
  "correlation_cap": null,
  "sector_cap": null,
  "day_trading": "unconstrained"
}
```

Sizing is fixed-fractional against the stop: the stop sits
`stop_atr_multiple` ATRs from entry, and the quantity is what loses
`risk_per_trade` of the account if it is hit, capped by
`max_position_fraction`. Volatility sets the size, not conviction. A
standard option contract is a lot of 100 units at multiplier one; two and a
half contracts is refused as `NotWholeLot`, not rounded. Arvo does not size
by Kelly; a Kelly fraction is a number to compare against, not a field.

Refusals, each named on the record: `Stale`, `TooSmall`, `NoStop`,
`NotWholeLot`, `DailyLossLimit`, `TooManyPositions`, `Correlated`, `Halted`.
Count them: a rule whose entries are mostly refused has a sizing problem
before it has an edge problem, and the advice says so.

**Correlation cap.** Rolling correlation of **returns, not levels** over the
last **90** bars, needing **30** paired observations before it speaks,
paired on time so a missing Tuesday is not compared with a Thursday. Rolling
because correlation breaks: two names that moved together in one regime
routinely stop. Five names each mildly correlated with the index and
almost perfectly with each other is one bet wearing five names.

**Warning band.** At four fifths of any limit the session says so — on its
row, in Operations, as a `warning` event — and keeps accepting. It is the
one moment a person can act before the gate does.

**Halts.** The drawdown halt is permanent for the run and lifts for nobody.
The kill switch is manual and lifts when a person releases it. A restart
lifts nothing. A session that starts against positions it did not open
adopts them and comes up halted until someone looks.

## Costs

Every experiment states commission (bps), slippage (bps), per-fill fee and
starting cash. Three tiers:

| Tier | Commission | Slippage | Fees |
|---|---|---|---|
| optimistic | half | as stated | as stated |
| realistic | as stated | as stated | as stated |
| conservative | ×1.5 | ×2, never under 5 bps | ×2 (and ×2 an option's spread) |

**A Supported verdict must survive the conservative tier**; the re-run can
only refuse. Fees as a share of *gross* return is reported when material —
gross, because comparing fees to a net figure understates them most where
it matters, when fees turned a positive result negative. Expectancy is
always "under which tier"; never quote it bare.

Live, slippage is **measured**: every fill against its decision price, with
what the finding assumed beside it, in the after-close review.

## Reading a result as a portfolio

- A **panel** averages members; a mean of drawdowns is a number that never
  happened. **Effective breadth** `N / (1 + (N−1)ρ̄)` over the *strategy's*
  returns says how many independent bets three agreeing results really are —
  reported to one decimal, never a gate.
- A **book** combines the curves as one account: equal weight, set once,
  never rebalanced (rebalancing for free is how backtests acquire returns),
  rescaled not re-simulated, so capital contention is invisible. Named,
  not hidden.
- **PSR** — the probability the true Sharpe exceeds zero given the sample's
  length, skew and tails — is on every curve. 1.2 from forty returns and
  1.2 from four thousand are not the same finding.

## Options: the stress floor

Option history reaches back to 2024 and has never seen 2008 or 2020. Every
option position is repriced at its entry as if that day were one of SPY's
worst, with implied volatility lifted to that day's VIX close:

| day | what | SPY | VIX |
|---|---|---|---|
| 2020-03-16 | covid crash | −10.9% | 82.7 |
| 2008-10-15 | financial crisis | −9.8% | 69.3 |
| 2011-08-08 | US downgrade | −6.5% | 48.0 |
| 2025-04-04 | tariff shock | −5.9% | 45.3 |
| 2022-09-13 | bear-market CPI day | −4.4% | 27.3 |
| 2018-02-05 | vol doubled, short-vol funds closed | −4.2% | 37.3 |

It is a **floor**: at-the-money VIX understates out-of-the-money put
volatility in a crash, the move lands at once with no exit, and levels are
not paths. A premium-selling rule whose worst backtested day is inside 2024
has not been shown the day that ends such rules.

## Live: is it still the rule?

A session's verdict — `Holding`, `Diverging`, `Inconclusive` — compares its
trades to the finding's out-of-sample expectation and **never touches the
gate**: a verdict is a judgement, a halt is a limit. It speaks after **10**
closed trades; drawdown past **1.5×** the finding's out-of-sample maximum
is Diverging at any count; the rule firing **4×** more or less often than
in the window (after 4 entries) is a regime change before it is a loss;
slippage is compared after **5** fills; **3** entries in a regime the finding
never traded in is a reason. `read_review` groups the day's losses by
condition, regime and exit reason, with what a person did beside the P&L —
intervening after losses is the most common way a Supported rule
underperforms its backtest.

**Promotion** to a real-money executor: a Supported verdict, **five days on
paper**, and no Diverging verdict on that paper session. In the engine, so
the window, the command line and an agent are all held to it.

## Do not

- Propose a size from conviction; the model sizes from ATR and risk per trade.
- Quote a return without its cost tier, or a Sharpe without its PSR.
- Read a panel's mean drawdown as a portfolio's.
- Call a stress number an estimate; it is a floor.
