---
name: arvo-rules
description: Write and fix Arvo rules, rulesets and universes. Use when asked to write a rule, add a strategy, define a parameter grid or search, build a universe, translate a Pine Script or TradingView strategy, set the risk model, or when write_rule/write_ruleset refuses something. Covers the rule JSON shape, the exact indicator and JSON Logic subset the engine runs, and the project folder layout.
---

# Rules, rulesets and universes

## Vocabulary

Say **rule** and **ruleset** to a person. `strategy` is the word in the code,
the protos, the MCP tool names and the findings — do not surface it in prose.

| | What it is | Where it lives |
|---|---|---|
| Rule | the logic: indicators, entry, exit, parameter defaults | `rules/<name>.json` |
| Ruleset | one rule with parameters fixed and the axes a study searches | `rulesets/<name>.json` |
| Configuration | one point of that grid, e.g. `fast: 10, slow: 30` | — |
| Universe | a named set of instruments a panel runs across | `universes/<name>.json` |
| Risk model | stops, sizing, limits — see `arvo-risk-and-costs` | the project's `risk.json` (`GetRiskModel` gives the path) |

The project folder is chosen in the window and remembered in
`%APPDATA%/com.arvo.desktop/project.json`; no env var. Paths are relative to
it. Arvo also ships compiled rules (`list_strategies`, e.g. `sma_cross`);
a rule as data is evaluated identically, to the cent.

## A rule

```json
{
  "name": "twin_cross",
  "label": "Moving-average crossover, as data",
  "premise": "The control, written down instead of compiled.",
  "interval": { "step": 1, "unit": "day" },
  "params": { "fast": 10, "slow": 30 },
  "indicators": {
    "fast": { "kind": "SMA", "period": "fast" },
    "slow": { "kind": "SMA", "period": "slow" }
  },
  "entry": { "cross_above": [ { "var": "fast" }, { "var": "slow" } ] },
  "exit":  { "cross_below": [ { "var": "fast" }, { "var": "slow" } ] }
}
```

**Indicators** (TA-Lib names; `input` is `open|high|low|close|volume`,
default `close`; `period` is a number or a parameter name, at most 10,000):

| kind | fields | notes |
|---|---|---|
| `SMA` | input, period | |
| `EMA` | input, period | seeded with the SMA of the first `period` values |
| `ATR` | period | true range, in price units |
| `RSI` | input, period | Wilder, 0–100; reports after `period + 1` bars |
| `MACD` | input, fast, slow, signal, line | `line` is `macd` (default), `signal` or `histogram`; **one declaration is one series** — declare it twice to compare line against signal |
| `MAX` | input (default `high`), period | highest over the last `period` bars, **including the current one** |
| `MIN` | input (default `low`), period | lowest over the last `period` bars, **including the current one** |

**Conditions** are JSON Logic, one operator per object, operands in a list.
An operand is `{"var": name}` — an indicator, or a bar field `open`, `high`,
`low`, `close`, `volume` — or a bare number.

| operator | meaning |
|---|---|
| `cross_above`, `cross_below` | fire **on the bar the relation changes**, not every bar it holds |
| `>`, `<` | strict only; there is no `>=`, `==` or arithmetic |
| `and`, `or`, `!` | an empty `and`/`or` is refused |

Long only, one instrument, one interval. Every number a ruleset's grid may
vary needs a default in `params`; a parameter named with no default is
refused when read. `write_rule` takes the definition as `rule` (an object
or its JSON) and refuses anything the engine would not run, naming the
construct (`unknown variant \`>=\`, expected one of …`) — the refusal is the
specification. What each
indicator is good for, and what common setups look like here, is
`arvo-indicators`.

## A ruleset

```json
{
  "name": "twin_grid",
  "label": "The twin, searched",
  "premise": "MSFT swings on a 20-30 day base",
  "interval": { "step": 1, "unit": "day" },
  "kind": { "kind": "grid", "rule": "twin_cross", "fixed": {}, "axes": { "fast": [5, 10], "slow": [20, 30] } }
}
```

**Keep the axes small.** The grid's size *is* the search size and the finding
is deflated against it: the winner of a sixteen-point grid must beat what the
best of sixteen no-skill tries would score. Widening a grid to find something
is what deflation exists to catch. `write_ruleset` replaces a ruleset of the
same name and refuses a name that collides with a compiled rule; then
`run_study` with `strategy` set to the ruleset's name. A rule as data has
no grid of its own: `run_study` on the bare rule is refused — *"the
parameter grid is empty, so the family tests nothing"* — so a study always
goes through a ruleset with at least one axis, even a one-value one.

## A universe

```json
{
  "name": "etf30",
  "reason": "Thirty of the most-traded US-listed ETFs by volume, as listed on 2026-09-26: liquidity, not returns.",
  "interval": { "step": 1, "unit": "day" },
  "since": "2016-01-01",
  "instruments": ["SPY.YF", "QQQ.YF", "IWM.YF"]
}
```

`reason` is required and must be a ground **other than returns** — an
index's members, a liquidity floor, a sector. A list chosen by looking at
what performed is the search the platform exists to deflate, and a universe
of today's index members is survivorship that no gate can see. Membership is
today's, not point-in-time; the finding says so. Members with no series yet
are named and left out. The engine keeps every member fetched every six
hours (`arvo-engine universes refresh` on demand).

## Translating Pine

`translate_pine` reads Pine v5 the way a person does, looking for the trade,
and refuses when unsure. Translated: `ta.sma`, `ta.ema`, `ta.atr`, `ta.rsi`,
`ta.highest`, `ta.lowest` on any bar field, **each assigned to a variable**
(an indicator called inline inside a condition is refused: "nothing in the
script gives this a value"); `ta.crossover`, `ta.crossunder`, `>`, `<`,
`and`, `or`, `not`; `input.*` as parameters with their defaults;
`strategy.entry` and `strategy.close` inside an `if`. Everything else —
`request.security`, `strategy.short`, `ta.stoch`, `ta.bb`, `ta.cross`
("either direction; say ta.crossover or ta.crossunder"), `>=` — is refused
**by name, all at once**. The `strategy(...)` settings line is listed as
ignored, like drawing. `plot` and its neighbours are set aside and listed,
never silently dropped. The interval comes from you, not the script.

It writes nothing: read the rule, then `write_rule`. Reading forty scripts
and keeping one is a search of forty that your findings are deflated
against; translate the script you meant to test.
