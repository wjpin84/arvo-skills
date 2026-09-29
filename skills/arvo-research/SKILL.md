---
name: arvo-research
description: Run and read Arvo studies honestly over the arvo MCP server. Use when asked to backtest, run a study, walk-forward, panel or universe test, rank or compare findings, read research memory, or judge whether a result is real. Also use when a verdict needs interpreting, when someone wants to re-run until something passes, or when a Supported result looks too good.
---

# Running a study in Arvo

The `arvo` MCP server is `arvo-mcp-server` from arvo-engine. It holds the
research token only: it **cannot fetch data and cannot trade**, enforced by
a build test that fails on any tool named `fetch`, `order`, `trade`, `share`,
`import`, `key` or `sign` ([ADR-0016](https://github.com/wjpin84/arvo-adrs)).
Fetching happens in the Arvo window. If data is missing, say so and stop.

`tools/list` is authoritative and verbose; read it rather than a list
written here. This skill is the order of operations and the epistemics.

Add the server with `--agent <name>`. Findings are saved under that author
and **deflated against everything that author has ever run**; without it
the loop is unbounded and uncounted.

## The rule that orders everything

> Anything that makes a wrong answer look right outranks anything that adds
> capability.

Most backtest results are noise; the job is to avoid believing them.

## What a study does

1. Runs every point of the grid on the first **70%** of the window (in
   sample) and selects the best.
2. Runs the winner alone on the held-out 30%. **Only out-of-sample numbers
   are the result.** In-sample numbers chose the configuration and are
   evidence of nothing.
3. Deflates: the winner's in-sample Sharpe must beat the expected best of
   that many no-skill tries.
4. Judges out of sample: **at least 30 round trips**, positive excess return
   over buy-and-hold of the same instrument at the same costs, drawdown
   **under 30%**.
5. Re-runs anything Supported at **conservative costs** — half again the
   commission, twice the slippage (never under 5 bps), twice the fees, twice
   an option's spread. This step can only refuse, never upgrade.
6. Records verdict, advice, both curves, every trade with its journal, the
   data's content hash, the ruleset's hash and the engine commit.

## The loop

1. `list_instruments` — what has data, at which resolution, over what range.
   Ids are `SYMBOL.VENUE` (`AAPL.RH`, `SPY.YF`).
2. `list_strategies`, `list_rules`, `list_rulesets` — a ruleset that cannot
   run is listed with the reason; fix that first.
3. Run **once**: `run_study` (seconds to a minute), `run_walk_forward`
   (slower by the number of folds), or `run_panel` (a universe, one
   parameter set for every member; minutes for a hundred names).
4. `open_finding` — read `read_this_first`, the verdict and the advice
   **before any number**.
5. `rank_findings` to place it; `compare_experiments` to explain why two
   differ.

## Verdicts and advice

`Supported`, `NotSupported`, `Inconclusive`. Inconclusive is the common one
and the honest one: too few trades, too short a window, a Sharpe whose
standard error swallows it. Report it as the result, not as a failure.

Advice is derived on read, never stored, and is **never advice about
markets**: every item is an instruction about the research — widen the
window, shrink the grid, check whether costs are the finding — with a
severity (`blocking`, `warning`, `note`) and its evidence line. A `blocking`
item means the numbers below are not a result yet. The blocking ones, in
order: the engine disagrees with itself (reconciliation failed — read
nothing); Supported only under the stated costs; too few trades; the search
explains the winner.

## Deflation: why re-running is not a strategy

Every run is your finding, deflated against every run you have made. The
winner of twenty studies is the best of a hundred and eighty draws even if
each was honestly deflated against its own nine. `compare_experiments`
deflates again — keeping the best of six is a search of six. A panel records
its universe's size as the search that keeping its best member would be.

So: state the hypothesis, run it once, report what came back. A
disappointing result is the finding. Every run is appended to
`agent-audit.jsonl` in the project.

Deflation cannot see survivorship: a universe of today's index members
passed no search and every gate passes it honestly.

## The leaderboard

`rank_findings` has one order: Supported under the conservative tier first,
by mean profit per closed out-of-sample trade under that tier; then
Supported under stated costs where the conservative tier was never measured;
then everything else whatever its return. Ties by drawdown, then trades. A
high return low in the list is low for a reason. Rows carry whether the data
has changed since — a **stale** finding is not evidence. Panels are not rows.

## Regimes

`inspect_regime` labels each bar after the fact. It says what the market was
doing, not what a rule could have known; cross-check it against the regime
on each trade in `open_finding`, never use it as a filter suggestion. The
definition is in `arvo-market-structure`.

## Do not

- Suggest a parameter change "to see if it helps" after reading a result;
  say what hypothesis a new grid tests, or stop.
- Report expectancy without the cost tier it is under.
- Read in-sample numbers as evidence, or a Sharpe without its PSR.
- Treat a panel, a comparison, or reading many Pine scripts as free.
