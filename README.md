# arvo-skills

Skills and agents for working with and on [Arvo](https://github.com/wjpin84/arvo-engine),
as a Claude Code plugin. Arvo-specific by design: every threshold, name and
rule here is read from the engine, not from general trading lore.

| | |
|---|---|
| `skills/arvo-research` | Running and reading studies over the `arvo` MCP server: what a study does, verdict before number, deflation, why re-running is not a strategy |
| `skills/arvo-rules` | Writing rules, rulesets and universes; the exact indicator and JSON Logic subset; translating Pine |
| `skills/arvo-indicators` | What each indicator the engine computes means, and the common setups as rule sketches — or why one cannot be expressed |
| `skills/arvo-market-structure` | Reading trend, range, volatility, regime, sessions, gaps and timeframes in Arvo's definitions; observation → hypothesis |
| `skills/arvo-market-data` | The library, venues, adjustment, the data-quality checks, comparing vendors, staleness, the broker feed vs the library |
| `skills/arvo-risk-and-costs` | The risk model, sizing and refusals, cost tiers, breadth and books, the options stress floor, live verdicts and promotion |
| `skills/arvo-platform` | Working across the repositories: where a change goes, branches, tests, releases, launching |
| `agents/arvo-study` | Tests one hypothesis end to end and reports the verdict, keeping the finding's JSON out of the main session |
| `agents/arvo-market` | Describes what an instrument is doing from the library and a read-only broker feed, in a context where no order, watchlist or scan-writing tool exists |

## Install

```
/plugin marketplace add wjpin84/arvo-skills
/plugin install arvo@arvo-skills
```

The research, market and study skills expect the `arvo` MCP server, named
for the author so the deflation counts the whole loop:

```
cargo build --release -p arvo-mcp-server      # in arvo-engine
claude mcp add arvo -- <path-to>/target/release/arvo-mcp-server --agent claude
```

It holds the research token only and cannot fetch or trade
([ADR-0016](https://github.com/wjpin84/arvo-adrs)). The market skills also
read from a broker MCP for live data when one is present; they never call
its order tools.

## Shape

The layout follows the open Agent Skills convention as
[agiprolabs/claude-trading-skills](https://github.com/agiprolabs/claude-trading-skills),
[SKE-Labs/agent-trading-skills](https://github.com/SKE-Labs/agent-trading-skills) and
[min9lin9/algo-trading-skills](https://github.com/min9lin9/algo-trading-skills)
use it: one folder per skill, a `SKILL.md` with `name` and `description`
frontmatter, a `.claude-plugin/` manifest. Skills stay under ~120 lines,
tables over prose, a `Do not` list instead of a mistakes section — the
SKE-Labs spec's conventions. Nothing is vendored from any of them.
