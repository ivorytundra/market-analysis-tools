# Market Analysis Tools

A Claude Code plugin for building reliable stock and options analysis tools.

## What it does

When you ask Claude to build market analysis tools, this skill guides it to produce production-quality code with guardrails that protect you from the expensive mistakes common in trading software:

- **Data validation** — every fetch is validated for schema, staleness, and completeness before use
- **Bias controls** — backtests include lookahead prevention, slippage modeling, and survivorship acknowledgment
- **Staleness UX** — timestamps on every data point, visible warnings when data is stale
- **Options correctness** — Greeks via battle-tested libraries (not hand-rolled), live risk-free rates, actual dividend yields
- **Partial failure handling** — never silently degrades; surfaces what failed and lets you decide

## Covers

- Stock/options screeners and scanners
- Technical indicator calculations (RSI, MACD, Bollinger, etc.)
- Options pricing and Greeks
- Backtesting frameworks
- Market data dashboards
- Options flow / unusual activity detectors
- Portfolio analytics and risk tools

## Install

```bash
claude plugin add ivorytundra/market-analysis-tools
```

Or clone and add locally:

```bash
git clone https://github.com/ivorytundra/market-analysis-tools.git
claude plugin add ./market-analysis-tools
```

## Structure

```
market-analysis-tools/
├── .claude-plugin/
│   └── plugin.json
├── skills/
│   └── market-analysis-tools/
│       ├── SKILL.md
│       └── references/
│           └── common-patterns.md
└── README.md
```

## Eval Results

Tested across 3 prompts over 2 iterations (with-skill vs baseline):

| Assertion | With Skill | Without |
|-----------|-----------|---------|
| Data staleness handling | 3/3 | 0/3 |
| Disclaimer present | 3/3 | 0/3 |
| Library-first for math | 3/3 | 1/3 |
| Tests included | 3/3 (139 total) | 0/3 |
| Data source abstraction | 3/3 | 0/3 |
| Partial failure handling | 3/3 | 0/3 |
| Pinned dependencies | 3/3 | 0/3 |

The skill costs ~2.5x more tokens but catches every guardrail the baselines miss.

## License

MIT
