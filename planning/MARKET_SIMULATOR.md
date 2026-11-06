# Market Simulator

How FinAlly generates believable, live-moving stock prices with no API key, no network and no external services. This is the default market data source (`PLAN.md` §6).

**Companions:** `MARKET_INTERFACE.md` defines the interface it implements.

**Status:** the simulator exists in `backend/app/market/simulator.py` and works. This document describes that approach and proposes small improvements. Measurements come from running the code. The listing at the end passes all existing simulator tests and the new contract tests in a scratch copy; it has not been applied to the repo.

## Goals

| Goal | How it's met |
|---|---|
| Prices look real | Geometric Brownian motion, realistic seed prices |
| Prices stay positive | GBM is multiplicative, so a price can't reach zero |
| Stocks differ | Per-ticker volatility and drift |
| Sectors move together | Correlated random draws via Cholesky decomposition |
| Occasional drama | Random 2-5% shocks |
| Any ticker works | Default parameters for symbols not in the seed list |
| Cheap | 200 tickers step in well under 1 ms |

## Structure

```
backend/app/market/
├── seed_prices.py    Data: seed prices, volatility, drift, correlation groups
└── simulator.py
    ├── GBMSimulator          Pure price model: no async, no I/O, no cache
    └── SimulatorDataSource   MarketDataSource adapter: runs the loop, writes to PriceCache
```

Keeping the model separate from the adapter matters: `GBMSimulator` can be tested by simply calling `step()`, with no event loop or sleeping.

```
SimulatorDataSource._run_loop  (every 500 ms)
    └─ GBMSimulator.step()
         ├─ Draw n independent normals
         ├─ Multiply by the Cholesky factor → correlated normals
         ├─ For each ticker: apply GBM, then maybe a shock
         └─ Return {ticker: rounded price}
    └─ PriceCache.update(ticker, price)  →  read by SSE, portfolio, trades
```

## The Model

### Geometric Brownian Motion

Each tick, every price is updated by:

```
S(t+dt) = S(t) · exp( (μ − σ²/2)·dt + σ·√dt·Z )
```

| Symbol | Meaning | Value |
|---|---|---|
| `S(t)` | Current price | Starts at the seed price |
| `μ` | Annual drift | 3-8% |
| `σ` | Annual volatility | 17-50% |
| `dt` | Tick length as a fraction of a trading year | 8.48×10⁻⁸ |
| `Z` | Correlated standard normal draw | Fresh each tick |

`dt` comes from 500 ms over a trading year of 252 days × 6.5 hours = 5,896,800 seconds, so `√dt` ≈ 2.91×10⁻⁴.

The `−σ²/2` term is the standard correction that makes `μ` the true expected growth rate. At this `dt`, drift is negligible on screen: AAPL's is about 0.00004 cents per tick. All visible movement comes from the `σ` term and from shocks.

### Parameters

| Ticker | Seed | σ | μ | Typical tick (1σ) | Flat ticks after rounding |
|---|---|---|---|---|---|
| AAPL | $190 | 0.22 | 0.05 | $0.012 | 32% |
| GOOGL | $175 | 0.25 | 0.05 | $0.013 | 30% |
| MSFT | $420 | 0.20 | 0.05 | $0.024 | 16% |
| AMZN | $185 | 0.28 | 0.05 | $0.015 | 26% |
| TSLA | $250 | 0.50 | 0.03 | $0.036 | 11% |
| NVDA | $800 | 0.40 | 0.08 | $0.093 | 4% |
| META | $500 | 0.30 | 0.05 | $0.044 | 9% |
| JPM | $195 | 0.18 | 0.04 | $0.010 | 36% |
| V | $280 | 0.17 | 0.04 | $0.014 | 27% |
| NFLX | $600 | 0.35 | 0.05 | $0.061 | 7% |
| Others | Generated | 0.25 | 0.05 | Varies | Varies |

Flat ticks are measured over 200,000 ticks with shocks off. A flat tick means `direction` is `flat` and the price doesn't flash, which makes the stream look more natural: cheap, calm stocks flash less often than volatile ones. The existing docstring's "sub-cent moves" is out of date: typical moves are 1-9 cents.

Over 200,000 ticks, measured annualized volatility matched the inputs to within 0.001 for every ticker.

Seed prices are illustrative and do not track real prices.

### Correlation

Real stocks don't move independently. To get correlated draws from independent ones:

1. Build a correlation matrix `C` from the pairwise rules below.
2. Factor it once: `C = L·Lᵀ` (Cholesky).
3. Each tick, draw independent normals `z` and use `L·z`.

| Pair | ρ | Measured (tick returns) |
|---|---|---|
| Tech with tech (AAPL, GOOGL, MSFT, AMZN, META, NVDA, NFLX) | 0.6 | AAPL-MSFT 0.602 |
| Finance with finance (JPM, V) | 0.5 | JPM-V 0.501 |
| TSLA with anything | 0.3 | TSLA-AAPL 0.298 |
| Cross-sector, or any unknown ticker | 0.3 | AAPL-JPM 0.298 |

TSLA is listed in the tech set, but a rule checked first gives it 0.3 with everything.

`L` is rebuilt only when tickers change. That cost 7.5 ms at 200 tickers, and adding a ticker is rare. The matrix stayed positive-definite at every size tested (up to 200), which Cholesky requires.

### Shock Events

After the GBM step, each ticker has a probability `event_probability` (default 0.001) of a shock:

```
S ← S · (1 ± U), where U ~ Uniform(0.02, 0.05) and the sign is 50/50
```

With 10 tickers at 2 ticks per second, that is one shock about every 50 seconds across the watchlist (measured: 52 seconds), or one per ticker about every 8 minutes.

### Shocks Dominate at the Default Setting

**This is the main thing to know when tuning the simulator.** Shocks are rare but enormous compared with a normal tick, around 300 times the variance for AAPL. Measured over 60 one-hour runs per setting:

| `event_probability` | Seconds between shocks (10 tickers) | Hourly move, 1σ: AAPL | JPM | TSLA | NVDA | Hourly correlation AAPL-MSFT |
|---|---|---|---|---|---|---|
| 0 (GBM only) | Never | 0.6% | 0.4% | 1.1% | 1.0% | 0.66 |
| 0.0001 | 500 | 3.0% | 3.4% | 2.8% | 3.4% | About 0 |
| 0.0002 | 250 | 5.0% | 4.7% | 4.8% | 4.6% | About 0 |
| 0.0005 | 100 | 6.7% | 5.4% | 6.2% | 7.6% | About 0 |
| **0.001 (default)** | **50** | **10.1%** | **9.7%** | **8.9%** | **11.0%** | **About 0** |

With 60 runs the correlation figures are noisy, roughly ±0.25, but the collapse is clear. The hourly standard deviation closely follows `√(ticks × p × 0.0013)`.

So, at the default:

- A sleepy bank moves as much as TSLA over an hour, because `σ` no longer matters at that timescale.
- Sector correlation is visible tick by tick but disappears over longer periods, because shocks are independent per ticker.
- A long-running container wanders far: about 27% (1σ) over 8 hours.
- A 30-minute demo gets about a 7% swing per ticker, which is what makes the P&L chart and heatmap interesting.

That last point is why this may be the right trade-off for a demo. See [Open Decisions](#open-decisions).

## Behaviour

### Lifecycle

| Call | Behaviour |
|---|---|
| `start(tickers)` | Builds the simulator, **seeds the cache immediately** so SSE has data on the first read, then starts the loop |
| `add_ticker(t)` | Adds it, rebuilds `L`, and seeds the cache before returning, so a buy can follow immediately |
| `remove_ticker(t)` | Removes it from the simulator and the cache, and rebuilds `L` |
| `stop()` | Cancels the loop. Safe to repeat |
| `get_end_of_day(tickers)` | Returns a synthetic prior session where open, high, low and close all equal the seed price |

An exception in `step()` is logged and the loop continues, so a bad tick can't end the price stream.

### Adding Unknown Tickers

Answering `PLAN.md` §13 item 3: any well-formed symbol is accepted, with default parameters (σ 0.25, μ 0.05) and correlation 0.3 with everything. It gets a pseudo-price between $50 and $300.

The current code picks that price with `random.uniform`, so PYPL started at $276 and then, after a restart, at $297 in one test. Deriving it from a hash of the symbol keeps it stable (see below). Sessions still restart from the seed either way.

### What the Simulator Doesn't Do

Each is a deliberate simplification:

- **No market hours.** Prices tick around the clock.
- **No overnight gaps, volume, bid/ask or order book.** Market orders fill at the last price.
- **No persistence.** A restart puts every ticker back at its seed price (see [Open Decisions](#open-decisions)).
- **No trends or volatility clustering.** Each tick is independent of the last.
- **Prices rounded to 2 decimals** for output, while the model keeps full precision internally, so rounding doesn't accumulate.

## Proposed Changes

Small, and the existing tests pass unchanged.

| Change | Problem today | Evidence |
|---|---|---|
| Normalize tickers with `normalize_ticker` | `add_ticker("aapl")` creates a second, separately priced ticker | Reproduced: `['AAPL', 'PYPL', 'aapl']` |
| Derive `dt` from `update_interval` | `update_interval=5.0` still simulates 0.5 seconds per tick, so slowing the loop also slows the market by 10× | Reproduced |
| Stable generated seeds via `seed_price_for()` | An added ticker gets a new random price on restart | Reproduced |
| Add `seed` and use one `np.random.Generator` | Results can't be reproduced, because the code mixes global `np.random` and `random` | Two seeded runs now match exactly over 50 steps |
| Pass `previous_close` to the cache | No daily change % | `MARKET_INTERFACE.md` |
| Implement `feed` and `get_end_of_day` | New interface methods | `MARKET_INTERFACE.md` |

The seeded generator also lets tests and E2E runs assert on specific prices.

## Testing

The existing suite covers `test_simulator.py` (19 tests) and `test_simulator_source.py` (10 tests). When changing the model, also check its statistics, using a fixed seed and shocks off:

```python
import math

import numpy as np
import pytest

from app.market.seed_prices import SEED_PRICES, TICKER_PARAMS
from app.market.simulator import GBMSimulator


def test_realized_volatility_and_correlation_match_inputs():
    tickers = list(SEED_PRICES)
    sim = GBMSimulator(tickers, event_probability=0.0, seed=7)
    prices = {t: [sim.get_price(t)] for t in tickers}
    for _ in range(50_000):
        sim.step()
        for t in tickers:
            prices[t].append(sim.get_price(t))

    returns = {t: np.diff(np.log(prices[t])) for t in tickers}
    for t in tickers:
        realized = returns[t].std() / math.sqrt(GBMSimulator.DEFAULT_DT)
        assert realized == pytest.approx(TICKER_PARAMS[t]["sigma"], rel=0.02)

    measured = np.corrcoef(returns["AAPL"], returns["MSFT"])[0, 1]
    assert measured == pytest.approx(0.6, abs=0.02)
```

To see it running: `cd backend && uv run market_data_demo.py` shows a live terminal dashboard.

## Open Decisions

1. **How dramatic should shocks be?** The default keeps the demo lively but erases per-stock character and sector behaviour beyond a few minutes. **Recommendation: lower to 0.0002**, one shock about every 4 minutes across the watchlist. It matches `PLAN.md`'s "occasional" better and halves the long-run wander, while a 30-minute demo still swings about 3%. Set 0.001 back if demos feel flat. It's one constant, and the listing below still has 0.001 until you decide.
2. **Should prices survive a restart?** Today, a restart resets prices to seeds while positions and average costs persist in SQLite, so P&L jumps. **Recommendation: accept it for now.** Fixing it means a new table and a save-on-shutdown path that isn't in the `PLAN.md` schema, and restarts are rare in a demo. If it becomes a problem, the small fix is an optional `initial_prices` argument on `start()`.

## Reference Implementation

### `seed_prices.py`

```python
"""Seed prices and per-ticker parameters for the market simulator."""

import hashlib

# Realistic starting prices for the default watchlist (as of project creation)
SEED_PRICES: dict[str, float] = {
    "AAPL": 190.00,
    "GOOGL": 175.00,
    "MSFT": 420.00,
    "AMZN": 185.00,
    "TSLA": 250.00,
    "NVDA": 800.00,
    "META": 500.00,
    "JPM": 195.00,
    "V": 280.00,
    "NFLX": 600.00,
}

# Per-ticker GBM parameters
# sigma: annualized volatility (higher = more price movement)
# mu: annualized drift / expected return
TICKER_PARAMS: dict[str, dict[str, float]] = {
    "AAPL": {"sigma": 0.22, "mu": 0.05},
    "GOOGL": {"sigma": 0.25, "mu": 0.05},
    "MSFT": {"sigma": 0.20, "mu": 0.05},
    "AMZN": {"sigma": 0.28, "mu": 0.05},
    "TSLA": {"sigma": 0.50, "mu": 0.03},  # High volatility
    "NVDA": {"sigma": 0.40, "mu": 0.08},  # High volatility, strong drift
    "META": {"sigma": 0.30, "mu": 0.05},
    "JPM": {"sigma": 0.18, "mu": 0.04},  # Low volatility (bank)
    "V": {"sigma": 0.17, "mu": 0.04},  # Low volatility (payments)
    "NFLX": {"sigma": 0.35, "mu": 0.05},
}

# Default parameters for tickers not in the list above (dynamically added)
DEFAULT_PARAMS: dict[str, float] = {"sigma": 0.25, "mu": 0.05}

# Correlation groups for the simulator's Cholesky decomposition
# Tickers in the same group have higher intra-group correlation
CORRELATION_GROUPS: dict[str, set[str]] = {
    "tech": {"AAPL", "GOOGL", "MSFT", "AMZN", "META", "NVDA", "NFLX"},
    "finance": {"JPM", "V"},
}

# Correlation coefficients
INTRA_TECH_CORR = 0.6  # Tech stocks move together
INTRA_FINANCE_CORR = 0.5  # Finance stocks move together
CROSS_GROUP_CORR = 0.3  # Between sectors / unknown tickers
TSLA_CORR = 0.3  # TSLA does its own thing


def seed_price_for(ticker: str) -> float:
    """Known seed price, or a stable pseudo-price in [50, 300) for any other symbol.

    Derived from the symbol so the same ticker restarts at the same price.
    """
    if ticker in SEED_PRICES:
        return SEED_PRICES[ticker]
    digest = hashlib.sha256(ticker.encode()).digest()
    return round(50.0 + int.from_bytes(digest[:4], "big") / 2**32 * 250.0, 2)
```

### `simulator.py`

```python
"""GBM-based market simulator."""

from __future__ import annotations

import asyncio
import logging
import math
from datetime import date, timedelta

import numpy as np

from .cache import PriceCache
from .interface import FeedKind, MarketDataSource, normalize_ticker
from .models import DailyBar
from .seed_prices import (
    CORRELATION_GROUPS,
    CROSS_GROUP_CORR,
    DEFAULT_PARAMS,
    INTRA_FINANCE_CORR,
    INTRA_TECH_CORR,
    TICKER_PARAMS,
    TSLA_CORR,
    seed_price_for,
)

logger = logging.getLogger(__name__)


class GBMSimulator:
    """Geometric Brownian Motion simulator for correlated stock prices.

    Math:
        S(t+dt) = S(t) * exp((mu - sigma^2/2) * dt + sigma * sqrt(dt) * Z)

    Where:
        S(t)   = current price
        mu     = annualized drift (expected return)
        sigma  = annualized volatility
        dt     = time step as fraction of a trading year
        Z      = correlated standard normal random variable

    The tiny dt (~8.5e-8 for 500ms ticks over 252 trading days * 6.5h/day)
    produces sub-cent moves per tick that accumulate naturally over time.
    """

    # 500ms expressed as a fraction of a trading year
    # 252 trading days * 6.5 hours/day * 3600 seconds/hour = 5,896,800 seconds
    TRADING_SECONDS_PER_YEAR = 252 * 6.5 * 3600  # 5,896,800
    DEFAULT_DT = 0.5 / TRADING_SECONDS_PER_YEAR  # ~8.48e-8

    def __init__(
        self,
        tickers: list[str],
        dt: float = DEFAULT_DT,
        event_probability: float = 0.001,
        seed: int | None = None,
    ) -> None:
        self._dt = dt
        self._event_prob = event_probability
        self._rng = np.random.default_rng(seed)  # Pass a seed for reproducible runs

        # Per-ticker state
        self._tickers: list[str] = []
        self._prices: dict[str, float] = {}
        self._params: dict[str, dict[str, float]] = {}

        # Cholesky decomposition of the correlation matrix (for correlated moves)
        self._cholesky: np.ndarray | None = None

        # Initialize all starting tickers
        for ticker in tickers:
            self._add_ticker_internal(ticker)
        self._rebuild_cholesky()

    # --- Public API ---

    def step(self) -> dict[str, float]:
        """Advance all tickers by one time step. Returns {ticker: new_price}.

        This is the hot path — called every 500ms. Keep it fast.
        """
        n = len(self._tickers)
        if n == 0:
            return {}

        # Generate n independent standard normal draws
        z_independent = self._rng.standard_normal(n)

        # Apply Cholesky to get correlated draws
        if self._cholesky is not None:
            z_correlated = self._cholesky @ z_independent
        else:
            z_correlated = z_independent

        result: dict[str, float] = {}
        for i, ticker in enumerate(self._tickers):
            params = self._params[ticker]
            mu = params["mu"]
            sigma = params["sigma"]

            # GBM: S(t+dt) = S(t) * exp((mu - 0.5*sigma^2)*dt + sigma*sqrt(dt)*Z)
            drift = (mu - 0.5 * sigma**2) * self._dt
            diffusion = sigma * math.sqrt(self._dt) * z_correlated[i]
            self._prices[ticker] *= math.exp(drift + diffusion)

            # Random event: ~0.1% chance per tick per ticker
            # With 10 tickers at 2 ticks/sec, expect an event ~every 50 seconds
            if self._rng.random() < self._event_prob:
                shock_magnitude = self._rng.uniform(0.02, 0.05)
                shock_sign = 1 if self._rng.random() < 0.5 else -1
                self._prices[ticker] *= 1 + shock_magnitude * shock_sign
                logger.debug(
                    "Random event on %s: %.1f%% %s",
                    ticker,
                    shock_magnitude * 100,
                    "up" if shock_sign > 0 else "down",
                )

            result[ticker] = round(self._prices[ticker], 2)

        return result

    def add_ticker(self, ticker: str) -> None:
        """Add a ticker to the simulation. Rebuilds the correlation matrix."""
        if ticker in self._prices:
            return
        self._add_ticker_internal(ticker)
        self._rebuild_cholesky()

    def remove_ticker(self, ticker: str) -> None:
        """Remove a ticker from the simulation. Rebuilds the correlation matrix."""
        if ticker not in self._prices:
            return
        self._tickers.remove(ticker)
        del self._prices[ticker]
        del self._params[ticker]
        self._rebuild_cholesky()

    def get_price(self, ticker: str) -> float | None:
        """Current price for a ticker, or None if not tracked."""
        return self._prices.get(ticker)

    def get_tickers(self) -> list[str]:
        """Return the list of currently tracked tickers."""
        return list(self._tickers)

    # --- Internals ---

    def _add_ticker_internal(self, ticker: str) -> None:
        """Add a ticker without rebuilding Cholesky (for batch initialization)."""
        if ticker in self._prices:
            return
        self._tickers.append(ticker)
        self._prices[ticker] = seed_price_for(ticker)
        self._params[ticker] = TICKER_PARAMS.get(ticker, dict(DEFAULT_PARAMS))

    def _rebuild_cholesky(self) -> None:
        """Rebuild the Cholesky decomposition of the ticker correlation matrix.

        Called whenever tickers are added or removed. O(n^2) but n < 50.
        """
        n = len(self._tickers)
        if n <= 1:
            self._cholesky = None
            return

        # Build the correlation matrix
        corr = np.eye(n)
        for i in range(n):
            for j in range(i + 1, n):
                rho = self._pairwise_correlation(self._tickers[i], self._tickers[j])
                corr[i, j] = rho
                corr[j, i] = rho

        self._cholesky = np.linalg.cholesky(corr)

    @staticmethod
    def _pairwise_correlation(t1: str, t2: str) -> float:
        """Determine correlation between two tickers based on sector grouping.

        Correlation structure:
          - Same tech sector:   0.6
          - Same finance sector: 0.5
          - TSLA with anything: 0.3 (it does its own thing)
          - Cross-sector:       0.3
          - Unknown tickers:    0.3
        """
        tech = CORRELATION_GROUPS["tech"]
        finance = CORRELATION_GROUPS["finance"]

        # TSLA is in tech set but behaves independently
        if t1 == "TSLA" or t2 == "TSLA":
            return TSLA_CORR

        if t1 in tech and t2 in tech:
            return INTRA_TECH_CORR
        if t1 in finance and t2 in finance:
            return INTRA_FINANCE_CORR

        return CROSS_GROUP_CORR


class SimulatorDataSource(MarketDataSource):
    """MarketDataSource backed by the GBM simulator.

    Runs a background asyncio task that calls GBMSimulator.step() every
    `update_interval` seconds and writes results to the PriceCache.
    """

    def __init__(
        self,
        price_cache: PriceCache,
        update_interval: float = 0.5,
        event_probability: float = 0.001,
        seed: int | None = None,
    ) -> None:
        self._cache = price_cache
        self._interval = update_interval
        self._event_prob = event_probability
        self._seed = seed
        self._sim: GBMSimulator | None = None
        self._task: asyncio.Task | None = None

    @property
    def feed(self) -> FeedKind:
        return "simulated"

    async def start(self, tickers: list[str]) -> None:
        symbols = [normalize_ticker(t) for t in tickers]
        self._sim = GBMSimulator(
            tickers=symbols,
            dt=self._interval / GBMSimulator.TRADING_SECONDS_PER_YEAR,
            event_probability=self._event_prob,
            seed=self._seed,
        )
        # Seed the cache with initial prices so SSE has data immediately
        for ticker in symbols:
            self._seed_cache(ticker)
        self._task = asyncio.create_task(self._run_loop(), name="simulator-loop")
        logger.info("Simulator started with %d tickers", len(tickers))

    async def stop(self) -> None:
        if self._task and not self._task.done():
            self._task.cancel()
            try:
                await self._task
            except asyncio.CancelledError:
                pass
        self._task = None
        logger.info("Simulator stopped")

    async def add_ticker(self, ticker: str) -> None:
        ticker = normalize_ticker(ticker)
        if self._sim:
            self._sim.add_ticker(ticker)
            # Seed cache immediately so the ticker has a price right away
            self._seed_cache(ticker)
            logger.info("Simulator: added ticker %s", ticker)

    async def remove_ticker(self, ticker: str) -> None:
        ticker = normalize_ticker(ticker)
        if self._sim:
            self._sim.remove_ticker(ticker)
        self._cache.remove(ticker)
        logger.info("Simulator: removed ticker %s", ticker)

    def get_tickers(self) -> list[str]:
        return self._sim.get_tickers() if self._sim else []

    async def get_end_of_day(self, tickers: list[str]) -> dict[str, DailyBar]:
        """Synthetic prior session: the seed price is yesterday's close."""
        session = _previous_weekday(date.today())
        bars = {}
        for ticker in tickers:
            symbol = normalize_ticker(ticker)
            seed = seed_price_for(symbol)
            bars[symbol] = DailyBar(symbol, session, seed, seed, seed, seed)
        return bars

    def _seed_cache(self, ticker: str) -> None:
        price = self._sim.get_price(ticker) if self._sim else None
        if price is not None:
            self._cache.update(ticker, price, previous_close=seed_price_for(ticker))

    async def _run_loop(self) -> None:
        """Core loop: step the simulation, write to cache, sleep."""
        while True:
            try:
                if self._sim:
                    prices = self._sim.step()
                    for ticker, price in prices.items():
                        self._cache.update(ticker=ticker, price=price)
            except Exception:
                logger.exception("Simulator step failed")
            await asyncio.sleep(self._interval)


def _previous_weekday(day: date) -> date:
    day -= timedelta(days=1)
    while day.weekday() >= 5:
        day -= timedelta(days=1)
    return day
```
