# arvo-skills

Skills and agents for working with, and on, [Arvo](https://github.com/wjpin84/arvo-engine)
— the financial research platform whose premise is that most backtest
results are noise and the job is to avoid believing them. Packaged as a
Claude Code plugin; the skills are plain `SKILL.md` files in the open
Agent Skills layout, so other agents that read that format can load them too.

Everything here is specific to Arvo. Every threshold, name and rule in these
files was read from the engine's source or its documentation, not from
general trading lore, and each skill says what it is *not* able to express
rather than approximating.

## Install

```
/plugin marketplace add wjpin84/arvo-skills
/plugin install arvo@arvo-skills
```

The research and market skills, and both agents, expect Arvo's MCP server
to be registered under the name `arvo`, with an author so the engine can
deflate the agent's whole loop as one search:

```
cargo build --release -p arvo-mcp-server        # in arvo-engine
claude mcp add arvo -- <path>/arvo-mcp-server --agent claude
```

That server holds the research token only. It reads findings, writes rules
and runs studies; it cannot fetch data, reach a broker or place an order, and
a build test in the engine keeps it that way
([ADR-0016](https://github.com/wjpin84/arvo-adrs)). The market skills also
read from a broker MCP for live data when one is present, and only through
its read tools.

## What is here

### Skills

| Skill | Use it when |
|---|---|
| [`arvo-research`](skills/arvo-research/SKILL.md) | running or reading a study, walk-forward or panel; ranking or comparing findings; judging whether a result is real; someone wants to re-run until something passes |
| [`arvo-rules`](skills/arvo-rules/SKILL.md) | writing a rule, ruleset or universe; translating Pine; `write_rule` refused something. The exact indicator and JSON Logic subset the engine runs |
| [`arvo-indicators`](skills/arvo-indicators/SKILL.md) | turning a trading idea into a rule; explaining what a condition means; a rule never fires or fires every bar; which compiled rule already covers an idea |
| [`arvo-market-structure`](skills/arvo-market-structure/SKILL.md) | describing what an instrument is doing — trend, range, volatility, regime, session, gaps, timeframes — in Arvo's definitions, and turning an observation into a testable premise |
| [`arvo-market-data`](skills/arvo-market-data/SKILL.md) | a result looks wrong; data may be missing or bad; two vendors disagree; a finding went stale; what an instrument id or a bar file means |
| [`arvo-risk-and-costs`](skills/arvo-risk-and-costs/SKILL.md) | sizing, stops, the risk model, why the gate refused an order, cost tiers, options stress, whether a live session still matches its finding, promotion to real money |
| [`arvo-platform`](skills/arvo-platform/SKILL.md) | changing code in any Arvo repository: which crate or repo a change belongs in, branches, tests, releases, launching the engine and window |

### Agents

A subagent exists only where a job produces output the main session should
not have to read, or needs a tool set the main session should not have.

| Agent | Job | Tools |
|---|---|---|
| [`arvo-study`](agents/arvo-study.md) | tests one stated hypothesis, once, and reports the verdict and advice — not the finding's JSON | the `arvo` research tools |
| [`arvo-market`](agents/arvo-market.md) | reads an instrument from the library and a live feed and returns a short description plus one premise a rule could test | `arvo` read tools and the broker MCP's `get_*` tools only — no order, watchlist or scan-writing tool exists in its context |

## What the skills hold to

- **Verdict before number.** A finding is read in the order the engine
  writes it: `read_this_first`, the verdict, the advice, then numbers, and
  only out-of-sample ones.
- **Every run is a search.** Findings are deflated against everything the
  author has run; re-running until something passes lowers the bar and the
  engine counts it. The skills never suggest "try another parameter".
- **Describe, never recommend.** Arvo's advice is about the research, not
  the market. The market skills return a description and a hypothesis;
  nothing here says buy or sell.
- **Say what cannot be expressed.** A Donchian breakout, a Bollinger band,
  a chart pattern: the skills name it as untestable here, and the compiled
  rule if one covers it, rather than writing a rule that trades zero times.

## What it is not

- Not a trading-knowledge library. For crypto, DeFi, ICT or chart-pattern
  skills see
  [agiprolabs/claude-trading-skills](https://github.com/agiprolabs/claude-trading-skills),
  [SKE-Labs/agent-trading-skills](https://github.com/SKE-Labs/agent-trading-skills) and
  [min9lin9/algo-trading-skills](https://github.com/min9lin9/algo-trading-skills),
  whose layout this follows and whose content is not vendored.
- Not an agent that trades. Starting a session is a person's decision, made
  with the control token; promotion to real money needs five paper days
  whoever asks.

## Writing a skill

One folder under `skills/`, a `SKILL.md` whose `name` matches the folder
and whose `description` says *when* to use it. Under ~120 lines, tables
over prose, a `Do not` list instead of a mistakes section. A number or a
name goes in only with the engine file it was read from; if the engine
changes, the skill is wrong, so cite it. An agent under `agents/` lists its
tools explicitly, and a broker tool must be a `get_*`. `main` takes pull
requests.

## Related

| | |
|---|---|
| [arvo-engine](https://github.com/wjpin84/arvo-engine) | the engine, the CLI and `arvo-mcp-server` |
| [arvo-engine-api](https://github.com/wjpin84/arvo-engine-api) | the protos the MCP tools are built over |
| [arvo-docs](https://wjpin84.github.io/arvo-docs/) | the product documentation the skills cite |
| [arvo-adrs](https://github.com/wjpin84/arvo-adrs) | decisions, one per file; nothing about deciding lives here |

## Licence

Apache-2.0, as the engine. The skills describe Arvo; nothing here is a
derivative of Zed or of NautilusTrader, so none of their licences reach it.
