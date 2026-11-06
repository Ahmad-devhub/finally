# Market Data Interface Design

The unified Python API FinAlly uses to get stock prices: Massive when `MASSIVE_API_KEY` is set, otherwise the simulator. Everything downstream of it is unaware of which one is running.

**Companions:** `MASSIVE_API.md` (provider facts) and `MARKET_SIMULATOR.md` (simulator internals).

**Status:** the architecture below already exists in `backend/app/market/` and should stay. This document proposes a focused set of changes, because research found that real-data mode doesn't work today and that some requirements in `PLAN.md` §13 are unmet.

**Verification:** the code here was built in a scratch copy of the backend (not yet applied to the repo). It passes ruff and 18 new tests on SDK 2.2.0 and 2.9.1, running the real SDK over HTTP against a local server imitating Starter, Developer and Basic plans. The existing simulator, cache and model tests (53) still pass unchanged. It has not run against the live API.

## Design at a Glance

```
                 create_market_data_source(cache)
                   reads MASSIVE_API_KEY
                     /                \
              key set                  no key
                  /                      \
   MassiveDataSource                 SimulatorDataSource
   feed: "snapshot" or "eod"         feed: "simulated"
                  \                      /
                   writes        writes
                    \                /
                     v              v
                      PriceCache (latest price per ticker)
                     /        |            \
                    v         v             v
             SSE stream   Portfolio     Trade execution
```

Three rules define it:

1. **Live prices are only read from `PriceCache`.** Nothing downstream calls a source for a live price, so request handlers never wait on a network call and never exhaust a rate limit.
2. **Sources share one interface.** Choosing a source happens in one place, the factory.
3. **End-of-day data goes through the source.** `get_end_of_day()` is on-demand and may use the network, because it is not a live price.

## What Changes

| | Today | Proposed | Why |
|---|---|---|---|
| **Massive client** | Fails on every plan | Works on all plans | `MASSIVE_API.md`, Defects section |
| **Plans without snapshots** | 403 forever, no prices | Falls back to end-of-day prices | Free keys are the most common kind |
| **Daily change %** | Only tick-to-tick `change_percent` | `previous_close` and `day_change_percent` | `PLAN.md` §13 item 5 |
| **End-of-day API** | None | `get_end_of_day()` | New requirement |
| **What's being shown** | Unknown to the app | `source.feed` | Delayed or stale data must be labelled |
| **Ticker normalization** | Massive only; the simulator accepts `aapl` as a second ticker | `normalize_ticker()` in both | Reproduced duplicate |
| **Unknown tickers** | Massive never prices them, silently | `UnknownTickerError` from `add_ticker` | `PLAN.md` §13 item 3 |
| **New ticker price** | Massive waits up to one poll | Priced before `add_ticker` returns | Buying right after adding works |
| **Removal and SSE** | `remove()` doesn't bump `version` | It does | Clients see the removal |
| **Stale write after removal** | An in-flight poll can re-add the ticker | Results filtered to active tickers | Race condition |
| **Hung request** | Can block `start()` indefinitely on SDK 2.2.0 | Hard deadline | Reproduced 30-second block with a 2-second timeout set |

Every existing method keeps its signature, except `PriceCache.update`, which gains an optional argument. Code following the usage in `MARKET_DATA_SUMMARY.md` keeps working.

## Public API

```python
from app.market import (
    DailyBar,                    # One completed session for one ticker
    FeedKind,                    # "simulated" | "snapshot" | "eod"
    MarketDataSource,            # Abstract interface
    PriceCache,                  # Latest prices: the read side
    PriceUpdate,                 # One ticker's price at a moment
    UnknownTickerError,          # Raised by add_ticker
    create_market_data_source,   # Factory
    create_stream_router,        # FastAPI SSE router
    normalize_ticker,            # " aapl " -> "AAPL", or UnknownTickerError
)
```

### Reading Prices

```python
update = cache.get("AAPL")        # PriceUpdate | None
price = cache.get_price("AAPL")   # float | None
everything = cache.get_all()      # dict[str, PriceUpdate]
```

`PriceUpdate` has two different notions of change. Using the wrong one is an easy mistake:

| Field | Compared with | Use for |
|---|---|---|
| `direction`, `change`, `change_percent` | Previous tick | Green/red price flash |
| `day_change`, `day_change_percent` | `previous_close` | The "daily change %" column |

`day_change` and `day_change_percent` are `None` until a baseline is known. The frontend should show a dash.

**Daily baseline by feed:**

| Feed | `previous_close` is |
|---|---|
| `snapshot` | Massive's `prevDay.c` |
| `eod` | The close of the session before the one displayed |
| `simulated` | The seed price, so daily change means "since the simulator started" |

`PLAN.md` §13 item 5 asked for a session-open baseline. This resolves it the same way for the simulator and uses the true previous close when real data exists.

### Managing Tickers

```python
try:
    await source.add_ticker("pypl")   # Normalized; priced before returning where possible
except UnknownTickerError as e:
    ...                               # Return 400 to the client

await source.remove_ticker("PYPL")    # No-op if absent
source.get_tickers()                  # ["AAPL", ...]
```

| Situation | Simulator | Massive |
|---|---|---|
| Malformed symbol | `UnknownTickerError` | `UnknownTickerError` |
| Well-formed, unknown to provider | Accepted, with a stable generated price | `UnknownTickerError` |
| Provider unreachable | n/a | Accepted with a warning; priced on the next successful poll |

### End-of-Day Data

```python
bars = await source.get_end_of_day(["AAPL", "MSFT"])
bars["AAPL"].close          # float
bars["AAPL"].session_date   # datetime.date
```

Returns the most recent completed session before today (US Eastern). Tickers with no data are omitted, and a provider failure returns `{}` rather than raising. On Massive it costs one request per uncached day regardless of how many tickers you ask for, and completed sessions are cached.

### Knowing What You're Showing

```python
source.feed  # "simulated" | "snapshot" | "eod"
```

| Feed | Meaning | Price cadence | Suggested label |
|---|---|---|---|
| `simulated` | Generated | 500 ms | `SIMULATED` |
| `snapshot` | Real, polled. Up to 15 minutes old on Starter and Developer | `MASSIVE_POLL_INTERVAL` | `LIVE` or `DELAYED`, per plan |
| `eod` | Real closing prices | Refreshed hourly; changes daily | `END OF DAY` |

`feed` can change once after `start()`, from `snapshot` to `eod`, so read it when rendering rather than caching it. A good place to expose it is `GET /api/health`.

The API can't tell Advanced from Developer, so `snapshot` doesn't promise real-time. If the UI needs that distinction, add a `MASSIVE_DELAYED=true` setting rather than guessing from timestamps, which look stale whenever the market is closed.

## Source Selection

| `MASSIVE_API_KEY` | Source | Then |
|---|---|---|
| Absent, empty or whitespace | Simulator | `feed` is `simulated` |
| Set, plan has snapshots | Massive | `feed` is `snapshot` |
| Set, plan lacks snapshots | Massive | First poll gets 403, `feed` becomes `eod`, and a warning explains it |
| Set, invalid | Massive | Errors are logged, no prices appear, and the app stays up |

**A failing Massive key never silently switches to the simulator.** Someone who configured a key expects real prices, and fake prices that look real are worse than a visibly empty or end-of-day screen, particularly in an app showing portfolio values. The warning tells them how to choose the simulator.

### Environment Variables

| Variable | Default | Meaning |
|---|---|---|
| `MASSIVE_API_KEY` | Empty | Selects Massive when set |
| `MASSIVE_POLL_INTERVAL` | `15` | Seconds between snapshot polls, minimum 1. Use 2-5 on Advanced. Paid plans have no request limit, but faster polling on a delayed plan gains little |

The 15-second default is a safe starting point, not a rate-limit requirement: any plan with snapshots is unlimited.

## Massive Modes

```
start()
  └─ poll snapshot ──┬─ 200 ──────────────► feed = snapshot, poll every interval
                     ├─ 403 NOT_AUTHORIZED ► feed = eod, refresh daily bars hourly
                     └─ other error ───────► stay in snapshot, retry next interval
```

**Price fallback:** `last_trade.price`, then `min.close`, then `day.close`, then `prev_day.close`. This is what makes Stocks Starter work. A row with none of these leaves the last good price in place.

**End-of-day mode** stays within 5 requests/minute by spacing calls 12.5 seconds apart and running in the background, so it never delays startup:

1. Walk back from yesterday to find the latest session. Normally 1 call on a weekday, more after weekends and holidays. Prices appear now.
2. Continue to the session before it, for `previous_close`. Daily change appears about 12.5 seconds later.
3. Each session's full-market result is cached. So `add_ticker` needs no request and can reject unknown symbols for free.

Yesterday is never cached when empty, because it may simply be unpublished. Older empty days are cached as market closures.

## Using It from the App

These are the integration points for the rest of the backend. The lifespan example was run; the other two are illustrative because the database layer doesn't exist yet.

### Startup and Shutdown

```python
from contextlib import asynccontextmanager

from fastapi import FastAPI

from app.market import PriceCache, create_market_data_source, create_stream_router


@asynccontextmanager
async def lifespan(app: FastAPI):
    cache = PriceCache()
    source = create_market_data_source(cache)
    await source.start(load_tracked_tickers())  # Watchlist plus open positions

    app.state.price_cache = cache
    app.state.market_source = source
    app.include_router(create_stream_router(cache))

    yield

    await source.stop()
```

### Which Tickers Are Tracked

**Tracked tickers must be `watchlist ∪ open positions`**, otherwise removing a held ticker from the watchlist freezes its price and breaks P&L (`PLAN.md` §13 item 2). The source doesn't know about positions, so this rule lives in the watchlist and trade code:

```python
async def remove_from_watchlist(ticker: str) -> None:
    db.delete_watchlist_entry(ticker)
    if not db.has_open_position(ticker):
        await source.remove_ticker(ticker)


async def on_position_closed(ticker: str) -> None:
    if not db.is_on_watchlist(ticker):
        await source.remove_ticker(ticker)
```

### Executing a Trade

One shared function for manual and AI trades (`PLAN.md` §13 item 15). Making sure a ticker is tracked also answers §13 item 8, whether you can trade something you aren't watching.

```python
async def execute_trade(ticker: str, side: str, quantity: float) -> Trade:
    symbol = normalize_ticker(ticker)         # May raise UnknownTickerError
    await source.add_ticker(symbol)           # No-op if tracked; may raise UnknownTickerError
    price = cache.get_price(symbol)
    if price is None:
        raise TradeRejected(f"No price available for {symbol} yet")
    ...
```

### SSE

`GET /api/stream/prices` is unchanged. Each ticker's payload gains three fields, and existing fields are untouched:

```json
{
  "AAPL": {
    "ticker": "AAPL",
    "price": 190.0,
    "previous_price": 190.01,
    "timestamp": 1793790676.57,
    "change": -0.01,
    "change_percent": -0.0053,
    "direction": "down",
    "previous_close": 190.0,
    "day_change": 0.0,
    "day_change_percent": 0.0
  }
}
```

Each event is a complete snapshot of all tickers, so a client should drop tickers that disappear from it.

## Error Handling

Sources log and carry on. No background failure ends the loop or reaches a request handler.

| Failure | Behaviour |
|---|---|
| 429, 5xx, network failure, timeout | Warning; keep last prices; retry next interval |
| 401 invalid key | Error each poll; no prices; app stays up |
| 403 on snapshot | One warning; switch to end-of-day |
| A row without a usable price | Keep that ticker's last price |
| Empty ticker list | No request is sent, because an empty list would fetch the whole market |

Prices can therefore go stale without any error reaching the UI, and a consumer can detect it by comparing `PriceUpdate.timestamp` with the current time. SSE connection status only reflects the link to our backend, not the upstream feed.

## Testing Requirements

**Never mock the SDK's response objects with `MagicMock`.** That is exactly how the current client passed 73 tests without ever working. Build them from real JSON, or serve real JSON over HTTP:

```python
from massive.rest.models import TickerSnapshot

from app.market.massive_client import extract_quote


def test_extract_quote_uses_nanosecond_trade_time():
    snap = TickerSnapshot.from_dict({
        "ticker": "AAPL",
        "lastTrade": {"p": 100.0, "t": 1_700_000_000_000_000_000},
        "min": {"c": 99.5, "t": 1_700_000_000_000},
        "updated": 1_700_000_000_000_000_000,
    })
    assert extract_quote(snap) == (100.0, 1_700_000_000.0)


def test_extract_quote_falls_back_to_minute_bar_on_starter():
    snap = TickerSnapshot.from_dict({
        "ticker": "AAPL",
        "min": {"c": 99.5, "t": 1_700_000_000_000},
    })
    assert extract_quote(snap) == (99.5, 1_700_000_000.0)
```

Inject a real client pointed at a local server to cover the HTTP layer, including the URL that defect #1 got wrong:

```python
source = MassiveDataSource(
    "test-key",
    cache,
    client=RESTClient(api_key="test-key", base=fake_server_url, retries=0),
)
```

The scratch suite covered the following. Keep these as the minimum:

- **Contract, run against both sources:** normalization, idempotent add and remove, `UnknownTickerError`, cache populated after `add_ticker`, `get_end_of_day` shape, `version` bump on removal, repeated `stop()`, JSON-serializable `to_dict()`.
- **Massive request:** the exact URL path requested, no call with an empty watchlist.
- **Plans:** Developer uses `lastTrade`; Starter uses the minute bar; Basic downgrades to `eod`, gets a baseline, and never requests today.
- **End-of-day:** walks over closed days, caches sessions, re-checks an empty yesterday, contains errors.
- **Failures:** 429 and 500 keep the last price; 401 neither crashes nor downgrades; a stalled call is bounded; a removed ticker isn't written back.
- **Factory:** whitespace-only key selects the simulator; `MASSIVE_POLL_INTERVAL` is honoured.

## Migration Checklist

In priority order. Steps 1-3 restore real-data mode.

1. **`pyproject.toml`:** require `massive>=2.9.1`, and add `tzdata` and `urllib3` as direct dependencies (the client imports `zoneinfo` data and `urllib3.exceptions`; slim images may not ship time zone data). Run `uv lock`.
2. **`models.py`, `cache.py`, `interface.py`:** apply the listings below.
3. **`massive_client.py`, `factory.py`, `__init__.py`:** apply the listings below.
4. **`simulator.py`, `seed_prices.py`:** see `MARKET_SIMULATOR.md`.
5. **Tests:** rewrite the 8 tests that depend on removed internals (7 in `test_massive.py`, 1 in `test_factory.py`), and add the suites above.
6. **Live check:** run the script in `MASSIVE_API.md` with a real key.
7. **Docs:** correct `PLAN.md` §6 (free tier polling) and `MARKET_DATA_SUMMARY.md` ("Complete"), and describe `feed` and `previous_close` in `backend/CLAUDE.md`.

## Open Decision

**Should a free Basic key show real closing prices that don't move, or use the simulator?** This design shows real prices with an `END OF DAY` label and a log message, on the principle that a configured key means "show me real data". The alternative is simpler: treat 403 as "use the simulator" and drop the mode switching and hourly refresh. It would look more alive, but would show invented prices to someone who supplied a real key, so it would at least need a prominent `SIMULATED` label.

## Reference Implementation

### `models.py`

```python
"""Data models for market data."""

from __future__ import annotations

import time
from dataclasses import dataclass, field
from datetime import date


@dataclass(frozen=True, slots=True)
class PriceUpdate:
    """Immutable snapshot of a single ticker's price at a point in time."""

    ticker: str
    price: float
    previous_price: float  # Previous *tick*: drives flash direction only
    timestamp: float = field(default_factory=time.time)  # Unix seconds
    previous_close: float | None = None  # Baseline for "daily change %"

    @property
    def change(self) -> float:
        return round(self.price - self.previous_price, 4)

    @property
    def change_percent(self) -> float:
        if self.previous_price == 0:
            return 0.0
        return round((self.price - self.previous_price) / self.previous_price * 100, 4)

    @property
    def direction(self) -> str:
        if self.price > self.previous_price:
            return "up"
        if self.price < self.previous_price:
            return "down"
        return "flat"

    @property
    def day_change(self) -> float | None:
        if not self.previous_close:
            return None
        return round(self.price - self.previous_close, 4)

    @property
    def day_change_percent(self) -> float | None:
        if not self.previous_close:
            return None
        return round((self.price - self.previous_close) / self.previous_close * 100, 4)

    def to_dict(self) -> dict:
        return {
            "ticker": self.ticker,
            "price": self.price,
            "previous_price": self.previous_price,
            "timestamp": self.timestamp,
            "change": self.change,
            "change_percent": self.change_percent,
            "direction": self.direction,
            "previous_close": self.previous_close,
            "day_change": self.day_change,
            "day_change_percent": self.day_change_percent,
        }


@dataclass(frozen=True, slots=True)
class DailyBar:
    """One completed trading session for one ticker."""

    ticker: str
    session_date: date
    open: float
    high: float
    low: float
    close: float
    volume: float | None = None
```

### `interface.py`

```python
"""Abstract interface for market data sources."""

from __future__ import annotations

import re
from abc import ABC, abstractmethod
from typing import Literal

from .models import DailyBar

FeedKind = Literal["simulated", "snapshot", "eod"]

_TICKER_RE = re.compile(r"^[A-Z][A-Z0-9.\-]{0,9}$")


class UnknownTickerError(ValueError):
    """The symbol is malformed, or the data provider says it does not exist."""


def normalize_ticker(ticker: str) -> str:
    """Canonical form used everywhere: stripped, upper-case, validated."""
    symbol = ticker.strip().upper()
    if not _TICKER_RE.match(symbol):
        raise UnknownTickerError(f"Invalid ticker symbol: {ticker!r}")
    return symbol


class MarketDataSource(ABC):
    """Contract for market data providers.

    Live prices are pushed into the shared PriceCache; downstream code reads
    the cache and never asks a source for a live price directly.
    """

    @property
    @abstractmethod
    def feed(self) -> FeedKind:
        """What the prices in the cache currently are. May change after start()."""

    @abstractmethod
    async def start(self, tickers: list[str]) -> None:
        """Begin producing updates. Must be called exactly once."""

    @abstractmethod
    async def stop(self) -> None:
        """Stop producing updates. Safe to call multiple times."""

    @abstractmethod
    async def add_ticker(self, ticker: str) -> None:
        """Track a ticker. No-op if tracked. Raises UnknownTickerError if invalid.

        Where possible the cache holds a price for it before this returns.
        """

    @abstractmethod
    async def remove_ticker(self, ticker: str) -> None:
        """Stop tracking a ticker and drop it from the cache. No-op if absent."""

    @abstractmethod
    def get_tickers(self) -> list[str]:
        """Currently tracked tickers."""

    @abstractmethod
    async def get_end_of_day(self, tickers: list[str]) -> dict[str, DailyBar]:
        """Most recent completed session's bar per ticker.

        Tickers with no data are omitted. May perform network I/O.
        """
```

### `massive_client.py`

```python
"""Massive (formerly Polygon.io) REST client for real market data."""

from __future__ import annotations

import asyncio
import json
import logging
from collections.abc import Callable
from datetime import date, datetime, timedelta
from typing import Any, TypeVar
from zoneinfo import ZoneInfo

from massive import RESTClient
from massive.exceptions import BadResponse
from massive.rest.models import GroupedDailyAgg, TickerSnapshot
from urllib3.exceptions import HTTPError

from .cache import PriceCache
from .interface import FeedKind, MarketDataSource, UnknownTickerError, normalize_ticker
from .models import DailyBar

logger = logging.getLogger(__name__)

T = TypeVar("T")

_MARKET_TZ = ZoneInfo("America/New_York")
_STOCKS = "stocks"  # Must be the plain string: the SDK formats it into the URL
_MAX_LOOKBACK_DAYS = 7  # Covers weekends plus any holiday cluster
_BASIC_CALL_SPACING = 12.5  # Seconds between calls: stays under 5 requests/minute
_REQUEST_ERRORS = (BadResponse, HTTPError, TimeoutError)


def _positive(value: float | None) -> float | None:
    """Massive omits fields, and a cleared bar can read 0: treat both as absent."""
    return value if value and value > 0 else None


def extract_quote(snap: TickerSnapshot) -> tuple[float, float | None] | None:
    """Best available (price, unix_seconds) from a snapshot, or None.

    Order matters: lastTrade is only present on plans that include trades, so
    Stocks Starter falls through to the minute bar.
    """
    updated = snap.updated / 1e9 if snap.updated else None  # nanoseconds

    trade = snap.last_trade
    if trade and _positive(trade.price):
        return trade.price, (trade.sip_timestamp / 1e9 if trade.sip_timestamp else updated)

    minute = snap.min
    if minute and _positive(minute.close):
        return minute.close, (minute.timestamp / 1e3 if minute.timestamp else updated)  # ms

    for bar in (snap.day, snap.prev_day):
        if bar and _positive(bar.close):
            return bar.close, updated
    return None


def _previous_close(snap: TickerSnapshot) -> float | None:
    return _positive(snap.prev_day.close) if snap.prev_day else None


def _error_status(error: BadResponse) -> str | None:
    """The SDK puts the raw body in the exception and drops the HTTP status."""
    try:
        return json.loads(str(error)).get("status")
    except ValueError:
        return None


def _to_bar(agg: GroupedDailyAgg, session: date) -> DailyBar:
    return DailyBar(
        ticker=agg.ticker,
        session_date=session,
        open=agg.open,
        high=agg.high,
        low=agg.low,
        close=agg.close,
        volume=agg.volume,
    )


class MassiveDataSource(MarketDataSource):
    """Polls the snapshot endpoint, or daily bars on plans without snapshots."""

    def __init__(
        self,
        api_key: str,
        price_cache: PriceCache,
        poll_interval: float = 15.0,
        eod_refresh_interval: float = 3600.0,
        request_timeout: float = 15.0,
        client: RESTClient | None = None,
    ) -> None:
        self._cache = price_cache
        self._interval = poll_interval
        self._eod_interval = eod_refresh_interval
        self._timeout = request_timeout
        # retries=0: the poll loop is the retry, and SDK retries would spend a
        # rate-limited plan's budget within a second.
        self._client = client or RESTClient(api_key=api_key, retries=0)
        self._tickers: list[str] = []
        self._task: asyncio.Task | None = None
        self._feed: FeedKind = "snapshot"
        self._sessions: dict[date, dict[str, DailyBar]] = {}  # {} = market was closed
        self._session_lock = asyncio.Lock()
        self._sessions_fetched = False

    @property
    def feed(self) -> FeedKind:
        return self._feed

    async def start(self, tickers: list[str]) -> None:
        self._tickers = [normalize_ticker(t) for t in tickers]
        await self._poll_snapshot()  # Also detects a plan without snapshots
        self._task = asyncio.create_task(self._run(), name="massive-poller")
        logger.info("Massive started: feed=%s tickers=%d", self._feed, len(self._tickers))

    async def stop(self) -> None:
        if self._task and not self._task.done():
            self._task.cancel()
            try:
                await self._task
            except asyncio.CancelledError:
                pass
        self._task = None

    async def add_ticker(self, ticker: str) -> None:
        symbol = normalize_ticker(ticker)
        if symbol in self._tickers:
            return
        if self._feed == "eod":
            await self._add_ticker_eod(symbol)
        else:
            await self._add_ticker_snapshot(symbol)
        self._tickers.append(symbol)

    async def remove_ticker(self, ticker: str) -> None:
        symbol = normalize_ticker(ticker)
        self._tickers = [t for t in self._tickers if t != symbol]
        self._cache.remove(symbol)

    def get_tickers(self) -> list[str]:
        return list(self._tickers)

    async def get_end_of_day(self, tickers: list[str]) -> dict[str, DailyBar]:
        try:
            sessions = await self._latest_sessions(1)
        except _REQUEST_ERRORS as e:
            logger.warning("Massive daily bars failed: %s", e)
            return {}
        latest = sessions[0] if sessions else {}
        symbols = [normalize_ticker(t) for t in tickers]
        return {s: latest[s] for s in symbols if s in latest}

    # --- Internal ---

    async def _call(self, fn: Callable[..., T], *args: Any, **kwargs: Any) -> T:
        """Run a blocking SDK call off the event loop, with a hard deadline."""
        return await asyncio.wait_for(asyncio.to_thread(fn, *args, **kwargs), self._timeout)

    async def _run(self) -> None:
        while True:
            if self._feed == "eod":
                await self._refresh_eod()
                await asyncio.sleep(self._eod_interval)
            else:
                await asyncio.sleep(self._interval)
                await self._poll_snapshot()

    async def _poll_snapshot(self) -> None:
        tickers = list(self._tickers)
        if not tickers:  # An empty list would request the entire market
            return
        try:
            snapshots = await self._call(self._client.get_snapshot_all, _STOCKS, tickers=tickers)
        except BadResponse as e:
            if _error_status(e) == "NOT_AUTHORIZED":
                logger.warning(
                    "This Massive plan has no snapshot access: showing end-of-day prices. "
                    "Unset MASSIVE_API_KEY to use the simulator for live-moving prices."
                )
                self._feed = "eod"
            else:
                logger.error("Massive poll rejected: %s", e)
            return
        except (HTTPError, TimeoutError) as e:  # 429, 5xx, network: retry next interval
            logger.warning("Massive poll failed: %s", e)
            return

        active = set(self._tickers)  # A ticker may have been removed mid-request
        for snap in snapshots:
            if snap.ticker in active:
                self._write_snapshot(snap)

    def _write_snapshot(self, snap: TickerSnapshot) -> bool:
        quote = extract_quote(snap)
        if quote is None:
            logger.debug("No usable price for %s", snap.ticker)
            return False  # Keep the last good price
        price, timestamp = quote
        self._cache.update(
            snap.ticker, price, timestamp=timestamp, previous_close=_previous_close(snap)
        )
        return True

    async def _add_ticker_snapshot(self, symbol: str) -> None:
        try:
            snap = await self._call(self._client.get_snapshot_ticker, _STOCKS, symbol)
        except BadResponse as e:
            raise UnknownTickerError(f"Massive rejected {symbol}: {_error_status(e)}") from e
        except (HTTPError, TimeoutError) as e:
            logger.warning("Could not verify %s now, next poll will price it: %s", symbol, e)
            return
        snap.ticker = symbol
        if not self._write_snapshot(snap):
            raise UnknownTickerError(f"Massive has no price for {symbol}")

    async def _add_ticker_eod(self, symbol: str) -> None:
        try:
            sessions = await self._latest_sessions(2)
        except _REQUEST_ERRORS as e:
            logger.warning("Could not verify %s now, next refresh will price it: %s", symbol, e)
            return
        if sessions and symbol not in sessions[0]:
            raise UnknownTickerError(f"Massive has no daily bar for {symbol}")
        self._write_eod(symbol, sessions)

    async def _refresh_eod(self) -> None:
        for count in (1, 2):  # Price first; the baseline may need a spaced-out call
            try:
                sessions = await self._latest_sessions(count)
            except _REQUEST_ERRORS as e:
                logger.warning("Massive daily bars failed: %s", e)
                return
            for symbol in list(self._tickers):
                self._write_eod(symbol, sessions)

    def _write_eod(self, symbol: str, sessions: list[dict[str, DailyBar]]) -> None:
        bar = sessions[0].get(symbol) if sessions else None
        if bar is None:
            return
        prior = sessions[1].get(symbol) if len(sessions) > 1 else None
        self._cache.update(symbol, bar.close, previous_close=prior.close if prior else None)

    async def _latest_sessions(self, count: int) -> list[dict[str, DailyBar]]:
        """Newest-first bars for the last `count` completed trading sessions.

        Starts from yesterday (US Eastern): Stocks Basic refuses the current
        day with a 403 at any hour, and on every plan today's bar is unfinished.
        """
        async with self._session_lock:
            found: list[dict[str, DailyBar]] = []
            today = datetime.now(_MARKET_TZ).date()
            for days_back in range(1, _MAX_LOOKBACK_DAYS + 1):
                session = today - timedelta(days=days_back)
                bars = self._sessions.get(session)
                if bars is None:
                    bars = await self._fetch_session(session)
                    # An empty yesterday may just be unpublished: ask again next time.
                    if bars or days_back > 1:
                        self._sessions[session] = bars
                if bars:
                    found.append(bars)
                    if len(found) == count:
                        break
            self._prune_sessions(today)
            return found

    async def _fetch_session(self, session: date) -> dict[str, DailyBar]:
        if self._feed == "eod" and self._sessions_fetched:
            await asyncio.sleep(_BASIC_CALL_SPACING)
        self._sessions_fetched = True
        aggs = await self._call(self._client.get_grouped_daily_aggs, session.isoformat())
        return {a.ticker: _to_bar(a, session) for a in aggs if a.ticker}

    def _prune_sessions(self, today: date) -> None:
        cutoff = today - timedelta(days=_MAX_LOOKBACK_DAYS)
        for session in [s for s in self._sessions if s < cutoff]:
            del self._sessions[session]
```

### `cache.py` and `factory.py` (changes)

```diff
--- a/cache.py
+++ b/cache.py
@@ -21,5 +21,11 @@
         self._version: int = 0  # Monotonically increasing; bumped on every update
 
-    def update(self, ticker: str, price: float, timestamp: float | None = None) -> PriceUpdate:
+    def update(
+        self,
+        ticker: str,
+        price: float,
+        timestamp: float | None = None,
+        previous_close: float | None = None,
+    ) -> PriceUpdate:
         """Record a new price for a ticker. Returns the created PriceUpdate.
 
@@ -37,4 +43,9 @@
                 previous_price=round(previous_price, 2),
                 timestamp=ts,
+                previous_close=(
+                    round(previous_close, 2)
+                    if previous_close is not None
+                    else (prev.previous_close if prev else None)
+                ),
             )
             self._prices[ticker] = update
@@ -60,5 +71,6 @@
         """Remove a ticker from the cache (e.g., when removed from watchlist)."""
         with self._lock:
-            self._prices.pop(ticker, None)
+            if self._prices.pop(ticker, None) is not None:
+                self._version += 1  # Lets SSE clients see the removal
 
     @property
--- a/factory.py
+++ b/factory.py
@@ -13,4 +13,18 @@
 logger = logging.getLogger(__name__)
 
+DEFAULT_POLL_INTERVAL = 15.0
+MIN_POLL_INTERVAL = 1.0
+
+
+def _poll_interval_from_env() -> float:
+    raw = os.environ.get("MASSIVE_POLL_INTERVAL", "").strip()
+    if not raw:
+        return DEFAULT_POLL_INTERVAL
+    try:
+        return max(float(raw), MIN_POLL_INTERVAL)
+    except ValueError:
+        logger.warning("Ignoring invalid MASSIVE_POLL_INTERVAL=%r", raw)
+        return DEFAULT_POLL_INTERVAL
+
 
 def create_market_data_source(price_cache: PriceCache) -> MarketDataSource:
@@ -26,5 +40,9 @@
     if api_key:
         logger.info("Market data source: Massive API (real data)")
-        return MassiveDataSource(api_key=api_key, price_cache=price_cache)
+        return MassiveDataSource(
+            api_key=api_key,
+            price_cache=price_cache,
+            poll_interval=_poll_interval_from_env(),
+        )
     else:
         logger.info("Market data source: GBM Simulator")
```

`__init__.py` additionally exports `DailyBar`, `FeedKind`, `UnknownTickerError` and `normalize_ticker`. The simulator's listing is in `MARKET_SIMULATOR.md`.
