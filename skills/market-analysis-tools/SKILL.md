---
name: market-analysis-tools
description: "Use when building stock/options analysis tools — screeners, signal detectors, dashboards, backtesting engines, options flow analyzers, or any tool that processes market data to surface trading insights. Trigger whenever your human partner mentions stocks, options, tickers, technical indicators, Greeks, screeners, scanners, backtesting, or market data pipelines."
---

# Market Analysis Tools

Build reliable stock and options analysis tools that your human partner can actually trust with real money decisions.

## Why This Skill Exists

Market analysis tools fail in specific, expensive ways: they silently return stale data, miscalculate Greeks, backtest with lookahead bias, or show a dashboard that looks right but isn't. A bug in a todo app wastes time. A bug in a trading tool loses money. Every design decision here must account for that asymmetry.

## When to Use

**Always:**
- Stock/options screeners and scanners
- Technical indicator calculations (RSI, MACD, Bollinger, etc.)
- Options pricing and Greeks (Black-Scholes, binomial, etc.)
- Backtesting frameworks
- Market data dashboards and visualizations
- Options flow and unusual activity detectors
- Portfolio analytics and risk tools

**Not this skill:**
- Execution/order management systems (too much regulatory surface area — flag and discuss)
- Crypto/DeFi tools (different data sources, different patterns)
- Pure portfolio tracking with no analysis logic

## The Iron Rules

```
1. NEVER TRUST RAW DATA — validate, timestamp, and flag staleness
2. NEVER BACKTEST WITHOUT BIAS CONTROLS — survivorship, lookahead, and slippage
3. NEVER DISPLAY A NUMBER WITHOUT ITS UNITS AND TIMESTAMP
4. NEVER HARDCODE MARKET ASSUMPTIONS — hours, holidays, settlement rules change
5. NEVER SILENTLY DEGRADE — partial data, failed fetches, and fallback values must be visible
```

## Delivery Checklist

Before presenting the tool to your human partner, verify:

- [ ] `requirements.txt` exists with pinned versions (not just package names)
- [ ] Tests exist AND are included in the output (not just referenced)
- [ ] CLI uses proper argument parsing (argparse/click/yargs), not `sys.argv` hacking
- [ ] Disclaimer is rendered in the UI/output, not just in README
- [ ] Every configurable value lives in config, not scattered in source files
- [ ] README includes: setup, usage example, sample output, and a troubleshooting section

## Architecture Checklist

Before writing code, your human partner should have answers to these. Ask if they don't:

1. **Data source** — Where does market data come from? (API, CSV, websocket, scraping)
2. **Freshness requirements** — Real-time, delayed 15m, end-of-day, historical only?
3. **Asset coverage** — Single ticker, watchlist, full-market scan?
4. **Output format** — Terminal, web dashboard, alerts, CSV export?
5. **Persistence** — In-memory only, SQLite, time-series DB?
6. **Cost sensitivity** — Free APIs (Yahoo Finance, FRED) vs. paid (Polygon, IEX, CBOE)?

## Data Layer

Market data is the foundation. Get this wrong and everything downstream is garbage.

### Data Source Selection

| Source | Cost | Coverage | Latency | Reliability |
|--------|------|----------|---------|-------------|
| Yahoo Finance (yfinance) | Free | Broad equities, options chains | 15m delay | Unstable, rate-limited |
| Alpha Vantage | Free tier | Equities, some options | 15m delay | Stable, 5 req/min free |
| Polygon.io | Paid | Full market, options | Real-time available | Production-grade |
| CBOE DataShop | Paid | Options-focused | Varies | Institutional quality |
| FRED | Free | Economic indicators | Daily | Very stable |

**Default recommendation for prototyping:** `yfinance` for equities, with explicit staleness checks. Move to Polygon or equivalent when the tool proves useful.

### Data Validation Pattern

Every data fetch must validate before downstream use:

```
fetch → validate schema → check timestamp → flag staleness → cache → serve
```

**Non-negotiable checks:**
- Is the response shaped correctly? (schema validation)
- Is the data from today / the expected trading session? (timestamp check)
- Is the market open or closed right now? (context flag)
- Are there gaps in the time series? (continuity check)
- Did the source return an error disguised as empty data? (null/empty guard)

If any check fails, surface the failure visibly — never silently serve bad data.

See `references/common-patterns.md` for the standard validation, staleness, and market status implementations. Use these as your starting point rather than reinventing them.

### Partial Data Failures

When a fetch partially succeeds (e.g., calls validate but puts don't, or 12 of 15 tickers return data), you must surface what failed:

- Log which specific items failed and why
- Display a visible warning: "Data incomplete: puts chain failed validation for AAPL"
- Never silently proceed with partial data as if it were complete
- Let your human partner decide whether partial results are useful

The temptation is to `except: pass` and show whatever worked. That's how people trade on incomplete data without knowing it.

### Staleness

Data displayed without a timestamp is data your human partner can't trust.

Every data point rendered to the user must show:
- **When** it was fetched (not just "live" — actual timestamp)
- **Market status** — pre-market, open, after-hours, closed
- **Delay** — "real-time", "15-min delayed", "end-of-day"

### Data Source Abstraction

Even when prototyping with yfinance, define a clean interface for data fetching so the source can be swapped later without rewriting downstream code. This doesn't need to be elaborate — a function signature or protocol class that returns a consistent data structure is enough. The point is that `screener.py` should not import `yfinance` directly.

## Technical Indicators

Indicators are math. Get the math right, then worry about presentation.

### Implementation Priority

1. **Use a battle-tested library first** — `pandas-ta`, `ta-lib`, or equivalent. Don't hand-roll RSI.
2. **If hand-rolling is required** — implement the canonical formula, cite the source, and validate against a known-good reference (e.g., compare your RSI output to TradingView for the same ticker/period).
3. **Test with known values** — for any indicator, find a published example with input data and expected output. Your test suite should reproduce it exactly (within floating-point tolerance).

### Common Indicator Pitfalls

| Indicator | Pitfall | Prevention |
|-----------|---------|------------|
| RSI | Using simple average instead of exponential (Wilder's smoothing) | Verify formula matches Wilder's original |
| MACD | Wrong EMA periods (12/26/9 is standard but not universal) | Make periods configurable, document defaults |
| Bollinger Bands | Using population vs. sample std dev | Use sample std dev (N-1) |
| Volume indicators | Comparing across different time frames without normalization | Always normalize to consistent intervals |
| Moving averages | Including current incomplete candle | Exclude current candle unless explicitly requested |

## Options-Specific Guidance

Options analysis has more ways to go wrong than equities. Respect the complexity.

### Greeks Calculation

- **Use a library** — `py_vollib`, `mibian`, or `QuantLib`. Rolling your own Black-Scholes is a rite of passage, not a production strategy.
- **Implied volatility** — This is solved iteratively (Newton-Raphson or bisection). If your IV calculation is a closed-form formula, it's wrong.
- **Interest rate** — Fetch the current risk-free rate from a live source (FRED DGS10 or similar). A hardcoded rate with a "user should update" comment is still a hardcoded rate — it will be wrong within months and nobody will remember to change it. If the live fetch fails, use a fallback value BUT display a visible warning that the rate is stale.
- **Dividends** — Fetch the current dividend yield for the underlying (yfinance provides this). Do not default to 0.0 — for dividend-paying stocks like AAPL (~1.5% yield), this makes call prices too high and put prices too low. If you can't fetch the yield, warn the user that results assume zero dividends.

### Options Chain Display

When displaying options chains:
- Show bid/ask spread, not just last price (last price can be hours old)
- Flag wide spreads (>10% of mid) — these are illiquid and the displayed price is misleading
- **Handle mid-price edge cases** — when mid = 0 (both bid and ask are zero), report spread as "N/A - no valid quote", not 0%. A zero spread implies tight liquidity; a zero mid implies no market.
- Show open interest alongside volume — high volume with low OI means new positions, not liquidity
- Display IV for each strike, not just ATM
- Include days to expiration as a number, not just the date

### Unusual Activity Detection

If building an options flow detector:
- Compare volume to open interest (V/OI ratio)
- Compare current volume to average volume (relative volume)
- Flag block trades vs. sweep orders if data source distinguishes them
- **Never claim to detect "smart money"** — label it as unusual activity and let your human partner interpret

## Backtesting

Backtesting is where most analysis tools lie to their users. The defaults must prevent the common lies.

### Bias Controls (Required)

| Bias | What It Is | Prevention |
|------|-----------|------------|
| **Lookahead** | Using data you wouldn't have had at decision time | Strict point-in-time data access — no peeking at future candles |
| **Survivorship** | Only testing on tickers that still exist today | Use historical constituent lists, or acknowledge the limitation |
| **Slippage** | Assuming fills at exact displayed price | Model slippage: at minimum, use next-bar open; better, add spread + impact |
| **Selection** | Running 1000 strategies and showing the best one | Track ALL strategies tested, report the full distribution |

### Lookahead Prevention

This is the most common and most dangerous bias. The rule is simple but easy to violate:

- **Signal** fires on bar[t] using only data from bar[t] and earlier
- **Execution** happens at bar[t+1] open (not bar[t] close, not bar[t+1] close)
- **Equity curve** must be built from the trade log, not recalculated from scratch — rebuilding creates a subtle risk of lookahead if signal logic and equity logic diverge

Write a test that asserts entry prices match next-bar opens. This is the single most valuable test in a backtesting engine.

### Backtest Output Requirements

A backtest result without context is a random number. Always show:
- **Total return** AND **annualized return**
- **Max drawdown** (peak-to-trough, with dates)
- **Sharpe ratio** (with the risk-free rate used, and a note that it's annualized)
- **Number of trades** (a strategy that trades once in 10 years is not a strategy)
- **Win rate** AND **average win/loss ratio** (win rate alone is meaningless)
- **Benchmark comparison** (SPY buy-and-hold at minimum)
- **Period tested** (with market regime context — was this all bull market?)

### The Backtest Honesty Check

Before showing results to your human partner, ask:
- Would this strategy have been possible to execute in real-time? (liquidity, execution speed)
- Are the returns concentrated in a few trades? (fragile)
- Does it only work in one market regime? (overfitted)
- Is the sample size large enough to be meaningful? (>30 trades minimum)

Surface these answers alongside the results, not hidden in footnotes.

## Dashboard & Visualization

### Display Principles

- **Red/green for direction** — universal, but also support colorblind modes (blue/orange)
- **Sparklines over tables** — a trend is worth a thousand numbers
- **Relative numbers over absolute** — "up 2.3%" is more useful than "$187.42" without context
- **Group related metrics** — price + volume together, Greeks together, risk metrics together

### Refresh & Staleness UX

- Show a visible "last updated" timestamp on every data panel
- If data is >5 minutes old during market hours, show a warning indicator
- If a refresh fails, show the failure — don't silently display stale data as current
- Provide manual refresh alongside any auto-refresh

## Multi-Ticker Performance

When scanning a watchlist of more than a few tickers, sequential fetching with rate-limit delays adds up fast (15 tickers × 1s delay = 15+ seconds). Consider:

- **Async fetching** with rate-limiting (e.g., `asyncio` with a semaphore, or `aiohttp` with per-second caps)
- **Progress indicator** — show which ticker is being scanned and how many remain
- **Fail-forward** — if one ticker fails, continue scanning the rest and report the failure at the end

For prototypes, sequential is fine. But if your human partner's watchlist grows past ~10 tickers, the UX degrades noticeably without concurrency.

## Common Rationalizations

| Excuse | Reality |
|--------|---------|
| "It's just a prototype, accuracy doesn't matter" | Prototypes that show wrong numbers build wrong intuitions. Your human partner will start making decisions based on what they see. |
| "I'll add slippage modeling later" | Backtest results without slippage are fantasy returns. Add it now or label results as "theoretical, no transaction costs." |
| "Yahoo Finance is good enough" | For prototyping, yes. For anything your human partner trades on, validate against a second source. |
| "The formula is simple, I don't need a library" | Black-Scholes is simple too. The edge cases (dividends, early exercise, extreme strikes) are where hand-rolled code breaks. |
| "I'll hardcode the risk-free rate for now" | "For now" means forever. What was 0.25% in 2020 was 5.25% in 2024. Fetch it live or make it a required config parameter that errors if not set. A hardcoded value with a "user should update" comment is still wrong. |
| "Survivorship bias doesn't matter for large caps" | It matters less, but it still matters. Acknowledge the limitation at minimum. |
| "Dividend yield doesn't matter much" | For AAPL at ~1.5% yield, ignoring dividends shifts theoretical call prices by several percent. Fetch the actual yield; don't default to 0.0. |
| "Partial data is fine, I'll just use what I got" | Your human partner doesn't know what's missing. Show them: "4 of 15 tickers failed, results are partial." Let them decide. |

## Red Flags — STOP and Reassess

| Signal | Problem |
|--------|---------|
| Displaying prices without timestamps | User can't assess freshness |
| Backtest showing >100% annual returns | Almost certainly has lookahead bias or no slippage |
| Options Greeks with hardcoded IV or rate | Results are wrong for any real use |
| Screener with no rate limiting | Will get IP-banned from data source |
| "Buy" / "Sell" signals without disclaimers | Liability risk, and the tool is overstepping its role |
| Dashboard with no error states | When the API goes down, user sees stale data and doesn't know it |
| `except: pass` around data fetches | Silent data loss — the most dangerous pattern in market tools |
| Spread percentage returning 0% when mid = 0 | Misleading — implies tight liquidity when there's actually no market |
| Equity curve rebuilt from scratch instead of trade log | Subtle lookahead risk if signal logic and curve logic diverge |
| Dividend yield defaulting to 0.0 without warning | Wrong Greeks for any stock that pays dividends |

## Disclaimer Requirement

Any tool that surfaces trading signals, screener results, or strategy backtests must include a visible disclaimer:

> This tool is for informational and educational purposes only. It does not constitute financial advice. Past performance does not indicate future results. Always do your own research before making investment decisions.

This isn't optional. Your human partner may share the tool. Protect them.

## Project Structure

```
market-tool/
├── src/
│   ├── data/           # Data fetching, validation, caching
│   ├── indicators/     # Technical indicator calculations
│   ├── options/        # Options-specific (Greeks, chains, pricing)
│   ├── backtest/       # Backtesting engine and bias controls
│   ├── screener/       # Screening/scanning logic
│   └── ui/             # Dashboard, charts, display
├── tests/              # Mirror src/ structure — MUST contain actual test files
├── config/             # API keys, market calendars, defaults
├── requirements.txt    # Pinned versions
└── docs/
    └── superpowers/
        └── specs/      # Design docs from brainstorming
```

Adapt to your human partner's preferences and existing project structure.

## Reference Code

Read `references/common-patterns.md` for standard implementations of:
- `ValidationResult` and `StaleDataWarning` dataclasses
- Market phase detection (pre-market, open, after-hours, closed)
- Standard `config/settings.py` layout
- Data fetcher interface pattern

These patterns were battle-tested across multiple tool types (screeners, backtests, dashboards). Start from them instead of reinventing — they handle the edge cases that trip up first-pass implementations.

## Integration with Other Skills

- **brainstorming** — Use first to nail down requirements (data source, asset coverage, output format)
- **test-driven-development** — Indicator math and backtest logic are perfect TDD targets: known inputs, expected outputs, zero ambiguity
- **writing-plans** — After brainstorming, plan the implementation in phases (data layer first, then analysis, then UI)
- **systematic-debugging** — When numbers look wrong, follow the debugging skill to trace from data source through calculation to display
