# Common Patterns for Market Analysis Tools

Standard implementations that every market analysis tool needs. Use these as your starting point — they handle edge cases that first-pass implementations miss.

## Table of Contents

1. [Validation & Staleness](#validation--staleness)
2. [Market Phase Detection](#market-phase-detection)
3. [Config Structure](#config-structure)
4. [Data Fetcher Interface](#data-fetcher-interface)

---

## Validation & Staleness

Every data fetch should return a result object that carries metadata about freshness and validity alongside the actual data.

```python
from dataclasses import dataclass, field
from datetime import datetime, timezone
from enum import Enum
from typing import Optional

import pandas as pd


class MarketPhase(Enum):
    PRE_MARKET = "Pre-Market"
    OPEN = "Open"
    AFTER_HOURS = "After-Hours"
    CLOSED = "Closed"
    WEEKEND = "Weekend"


@dataclass
class FetchMeta:
    """Metadata attached to every data fetch result."""
    fetched_at: datetime  # UTC timestamp of when data was retrieved
    source: str  # e.g., "yfinance", "polygon"
    delay_label: str  # e.g., "15-min delayed", "real-time", "end-of-day"
    market_phase: MarketPhase
    is_stale: bool = False  # True if data exceeds staleness threshold during market hours
    warnings: list[str] = field(default_factory=list)  # Partial failures, fallback values, etc.


class ValidationError(Exception):
    """Raised when fetched data fails validation checks."""
    pass


def validate_dataframe(
    df: pd.DataFrame,
    required_columns: set[str],
    label: str = "data",
) -> pd.DataFrame:
    """
    Validate a DataFrame from a market data source.

    Checks: not None, not empty, required columns present, not all-NaN.
    Raises ValidationError with a specific message on failure.
    """
    if df is None:
        raise ValidationError(f"{label}: received None instead of DataFrame")

    if df.empty:
        raise ValidationError(f"{label}: DataFrame is empty — source may have returned an error as empty data")

    missing = required_columns - set(df.columns)
    if missing:
        raise ValidationError(f"{label}: missing required columns: {missing}")

    # Detect columns that are entirely NaN — often signals a silent API failure
    all_nan_cols = [col for col in required_columns if df[col].isna().all()]
    if all_nan_cols:
        raise ValidationError(f"{label}: columns are entirely NaN (possible data source failure): {all_nan_cols}")

    return df


def check_staleness(
    fetched_at: datetime,
    market_phase: MarketPhase,
    threshold_seconds: int = 300,
) -> bool:
    """
    Returns True if data is stale (older than threshold during market hours).
    Data is never considered stale when the market is closed or on weekends.
    """
    if market_phase in (MarketPhase.CLOSED, MarketPhase.WEEKEND):
        return False

    age = (datetime.now(timezone.utc) - fetched_at).total_seconds()
    return age > threshold_seconds
```

### Usage Pattern

```python
# In your data fetcher:
meta = FetchMeta(
    fetched_at=datetime.now(timezone.utc),
    source="yfinance",
    delay_label="15-min delayed",
    market_phase=get_market_phase(),
)

try:
    df = validate_dataframe(raw_df, {"open", "high", "low", "close", "volume"}, label=f"{ticker} OHLCV")
except ValidationError as e:
    meta.warnings.append(str(e))
    # Surface this to the user — do NOT silently continue

meta.is_stale = check_staleness(meta.fetched_at, meta.market_phase)
if meta.is_stale:
    meta.warnings.append(f"Data is older than {threshold}s during market hours")
```

---

## Market Phase Detection

All three test runs independently built this same function. Use it as-is and configure the hours through your settings module.

```python
from datetime import datetime, time
import pytz


def get_market_phase(
    now: Optional[datetime] = None,
    tz: str = "US/Eastern",
    pre_market_open: time = time(4, 0),
    market_open: time = time(9, 30),
    market_close: time = time(16, 0),
    after_hours_close: time = time(20, 0),
) -> MarketPhase:
    """
    Determine current US market phase.

    All times are in Eastern Time by default.
    Does NOT handle market holidays — see note below.
    """
    eastern = pytz.timezone(tz)
    if now is None:
        now = datetime.now(eastern)
    else:
        now = now.astimezone(eastern)

    # Weekend check
    if now.weekday() >= 5:
        return MarketPhase.WEEKEND

    current_time = now.time()

    if current_time < pre_market_open:
        return MarketPhase.CLOSED
    elif current_time < market_open:
        return MarketPhase.PRE_MARKET
    elif current_time < market_close:
        return MarketPhase.OPEN
    elif current_time < after_hours_close:
        return MarketPhase.AFTER_HOURS
    else:
        return MarketPhase.CLOSED
```

**Market holidays:** This function does not detect holidays (MLK Day, Good Friday, etc.). For a prototype, document this limitation. For production, use the `exchange_calendars` or `pandas_market_calendars` package, which maintains accurate holiday schedules for major exchanges.

---

## Config Structure

Every tool needs a settings module. This layout was consistent across all test runs — use it as your template.

```python
"""
config/settings.py — All configurable values in one place.

Override via environment variables or CLI flags, never by editing this file in production.
"""

# --- Data Source ---
DATA_SOURCE = "yfinance"
DATA_DELAY_LABEL = "15-min delayed"
RATE_LIMIT_DELAY_SECONDS = 1.0  # Delay between API calls to avoid bans

# --- Staleness ---
STALENESS_THRESHOLD_SECONDS = 300  # 5 minutes during market hours
STALENESS_WARN_IN_UI = True  # Show warning badge when data is stale

# --- Market Hours (Eastern Time) ---
MARKET_TIMEZONE = "US/Eastern"
PRE_MARKET_OPEN = "04:00"
MARKET_OPEN = "09:30"
MARKET_CLOSE = "16:00"
AFTER_HOURS_CLOSE = "20:00"

# --- Options ---
WIDE_SPREAD_THRESHOLD = 0.10  # Flag bid/ask spreads wider than 10% of mid
MIN_OPEN_INTEREST = 10  # Ignore strikes with fewer contracts

# --- Risk-Free Rate ---
# Fetched live from FRED DGS10. Fallback used ONLY if fetch fails,
# and a warning is displayed to the user.
RISK_FREE_RATE_FRED_SERIES = "DGS10"
RISK_FREE_RATE_FALLBACK = 0.045  # Updated: 2024-Q4. MUST display warning if used.

# --- Backtesting ---
DEFAULT_SLIPPAGE_BPS = 5  # 5 basis points per side
MIN_TRADES_FOR_SIGNIFICANCE = 30

# --- Display ---
COLORBLIND_MODE = False  # True: blue/orange instead of red/green
```

**Key principle:** Every value that could change (rates, thresholds, hours, API behavior) lives here. Source files import from config — they never define their own magic numbers.

---

## Data Fetcher Interface

Even for a prototype, keeping the data source behind an interface prevents yfinance imports from spreading through your codebase. When your human partner outgrows yfinance, you swap one file instead of rewriting the screener, backtest engine, and dashboard.

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import Optional

import pandas as pd


@dataclass
class FetchResult:
    """Standard return type for all data fetchers."""
    data: Optional[pd.DataFrame]  # None if fetch failed entirely
    meta: FetchMeta  # Always present, even on failure


class MarketDataFetcher(ABC):
    """Interface for market data providers."""

    @abstractmethod
    def fetch_ohlcv(self, ticker: str, period: str = "1y", interval: str = "1d") -> FetchResult:
        """Fetch OHLCV price data."""
        ...

    @abstractmethod
    def fetch_options_chain(self, ticker: str, expiration: Optional[str] = None) -> FetchResult:
        """Fetch options chain. Returns calls and puts in the DataFrame."""
        ...

    @abstractmethod
    def fetch_quote(self, ticker: str) -> FetchResult:
        """Fetch current quote (price, volume, market cap)."""
        ...


class YFinanceFetcher(MarketDataFetcher):
    """yfinance implementation. Use for prototyping."""

    def fetch_ohlcv(self, ticker, period="1y", interval="1d"):
        # Implementation here — wrap yfinance calls with validation
        ...
```

**The point is not abstraction for its own sake.** The point is that `screener.py` and `backtest/engine.py` should never `import yfinance`. They receive a `FetchResult` and work with validated DataFrames. When the data source changes, only the fetcher implementation changes.

---

## Bid/Ask Spread Calculation

Handle the edge case that all three test runs got slightly wrong:

```python
def spread_percentage(bid: float, ask: float) -> Optional[float]:
    """
    Calculate bid/ask spread as percentage of mid-price.

    Returns None when mid-price is zero (no valid market).
    This prevents reporting 0% spread (implying tight liquidity)
    when there's actually no market for the contract.
    """
    if bid <= 0 and ask <= 0:
        return None  # No valid quote

    mid = (bid + ask) / 2
    if mid <= 0:
        return None  # No valid mid-price

    return (ask - bid) / mid
```
