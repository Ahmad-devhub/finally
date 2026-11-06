# Massive API Reference (formerly Polygon.io)

How to get real-time and end-of-day US stock prices for multiple tickers from Massive, written for FinAlly.

**Researched:** 2026-11-04, from massive.com docs and knowledge base, plus the source of the `massive` Python SDK (2.2.0, locked in `backend/uv.lock`; 2.9.1, latest).

**Verification level:** SDK behaviour was confirmed by running the real SDK against a local HTTP server that returns Massive's documented payloads. **Nothing here has run against the live API with a valid key, because no key was available.** Items that still need a live check are in [Live verification](#live-verification).

## Key Facts for FinAlly

1. **One call covers every ticker.** `GET /v2/snapshot/locale/us/markets/stocks/tickers?tickers=AAPL,MSFT,...` returns all requested tickers in one request.
2. **The free plan has no snapshots.** Stocks Basic gets end-of-day data only, at 5 requests/minute. Polling snapshots every 15 seconds on a free key returns 403 every time, so the "free tier: poll every 15s" guidance in `PLAN.md` §6 cannot work.
3. **"Real-time" costs $199/month.** Starter and Developer snapshots are delayed 15 minutes.
4. **Starter snapshots have no `lastTrade`.** Read the minute bar (`min.c`) instead on that plan.
5. **Trade timestamps are nanoseconds.** Bar timestamps are milliseconds.
6. **The current backend client cannot get prices on any plan.** See [Defects in the current code](#defects-in-the-current-code).

## Plans

Prices are from Massive's docs and comparison pages, which cite October 2026. Check [the pricing page](https://massive.com/pricing) before buying. Plans are per asset class, and individual plans are marked "Individual use only"; showing data to customers requires a Business plan.

| | Basic | Starter | Developer | Advanced | Business |
|---|---|---|---|---|---|
| **Price/month** | Free | $29 | $79 | $199 | $2,499 |
| **Request limit** | 5/minute | Unlimited | Unlimited | Unlimited | Unlimited |
| **Snapshots** | No | 15 min delayed | 15 min delayed | Real-time | Real-time |
| **`lastTrade` in snapshot** | n/a | No | 15 min delayed | Real-time | Expansion add-on |
| **`lastQuote` in snapshot** | n/a | No | No | Real-time | See docs |
| **Daily and custom bars** | End-of-day | 15 min delayed | 15 min delayed | Real-time | Real-time |
| **Bar history** | 2 years | 5 years | 10 years | All | All |
| **Previous-day bar, market status, ticker overview** | Yes | Yes | Yes | Yes | Yes |

What this means for FinAlly:

| You have | What the app can show | Recommendation |
|---|---|---|
| No key | Simulated, moving prices | **Default for most users** |
| Basic | Real closing prices that do not tick | Fine for checking real valuations; dull as a live demo |
| Starter or Developer | Real prices, about 15 minutes old | Good, but label them as delayed |
| Advanced | Real-time prices | Best, poll every 2-5 seconds |

## Setup

```bash
cd backend
uv add "massive>=2.9.1"   # 2.2.0 ignores its timeouts: see SDK gotchas
```

```python
import os

from massive import RESTClient

client = RESTClient(api_key=os.environ["MASSIVE_API_KEY"])
```

- **Base URL:** `https://api.massive.com`. The legacy `https://api.polygon.io` is still supported for an extended period.
- **Auth:** `Authorization: Bearer <key>` header (the SDK does this) or `?apiKey=<key>` query parameter.
- **Python:** 3.10 or later for current SDK versions.
- **Useful constructor options:** `retries` (default 3), `connect_timeout` and `read_timeout` (default 10.0), `base`, `pagination`, `trace`. Point `base` at a local server for tests.

The SDK is synchronous. Inside FastAPI, call it through `asyncio.to_thread`.

Plain HTTP works too:

```bash
curl -H "Authorization: Bearer $MASSIVE_API_KEY" \
  "https://api.massive.com/v2/snapshot/locale/us/markets/stocks/tickers?tickers=AAPL,MSFT"
```

## Real-Time Prices

### Full Market Snapshot (FinAlly's polling endpoint)

`GET /v2/snapshot/locale/us/markets/stocks/tickers`

| Query parameter | Notes |
|---|---|
| `tickers` | Comma-separated and case-sensitive. **An empty value returns the whole market (10,000+ rows).** |
| `include_otc` | Default `false` |

```python
snapshots = client.get_snapshot_all("stocks", tickers=["AAPL", "MSFT", "NVDA"])

for snap in snapshots:
    # Fields can be absent, so check before reading: see the fallback order below.
    trade_price = snap.last_trade.price if snap.last_trade else None
    minute_close = snap.min.close if snap.min else None
    previous_close = snap.prev_day.close if snap.prev_day else None
    print(snap.ticker, trade_price, minute_close, previous_close, snap.todays_change_percent)
```

Documented response:

```json
{
  "count": 1,
  "status": "OK",
  "tickers": [
    {
      "ticker": "BCAT",
      "day": {"o": 20.64, "h": 20.64, "l": 20.506, "c": 20.506, "v": 37216, "vw": 20.616},
      "prevDay": {"o": 20.79, "h": 21, "l": 20.5, "c": 20.63, "v": 292738, "vw": 20.6939},
      "min": {"av": 37216, "o": 20.506, "h": 20.506, "l": 20.506, "c": 20.506, "v": 5000, "vw": 20.5105, "n": 1, "t": 1684428600000},
      "lastTrade": {"p": 20.506, "s": 2416, "x": 4, "i": "71675577320245", "c": [14, 41], "t": 1605192894630916600},
      "lastQuote": {"P": 20.6, "S": 22, "p": 20.5, "s": 13, "t": 1605192959994246100},
      "todaysChange": -0.124,
      "todaysChangePerc": -0.601,
      "updated": 1605192894630916600
    }
  ]
}
```

| JSON field | SDK attribute | Meaning | Availability |
|---|---|---|---|
| `ticker` | `snap.ticker` | Symbol | Always |
| `lastTrade.p` | `snap.last_trade.price` | Latest trade price | Developer and up |
| `lastTrade.t` | `snap.last_trade.sip_timestamp` | **Nanoseconds** | Developer and up |
| `min.c` | `snap.min.close` | Latest minute bar's close | Starter and up |
| `min.t` | `snap.min.timestamp` | **Milliseconds** | Starter and up |
| `day.c` | `snap.day.close` | Current session so far | Starter and up |
| `prevDay.c` | `snap.prev_day.close` | **Previous close: the baseline for daily change** | Starter and up |
| `todaysChange`, `todaysChangePerc` | `snap.todays_change`, `snap.todays_change_percent` | Change from previous day | Starter and up |
| `updated` | `snap.updated` | **Nanoseconds** | Starter and up |
| `lastQuote.p`, `lastQuote.P` | `snap.last_quote.bid_price`, `.ask_price` | NBBO bid and ask | Advanced |

**Fallback order for "the current price":** `last_trade.price`, then `min.close`, then `day.close`, then `prev_day.close`. Ignore absent or zero values. Massive omits missing fields rather than returning null.

**Daily reset:** the docs say snapshot data is cleared at 3:30 AM EST and repopulates from about 4:00 AM EST. The SDK docstring says 12 AM. Don't rely on either time: keep the last good price whenever a row has no usable value.

### Unified Snapshot

`GET /v3/snapshot`, the same plans as above. It takes up to 250 tickers per request, and a 251-ticker request returns HTTP 400.

Unlike v2, it reports bad tickers explicitly: unknown symbols come back as rows with `error: "NOT_FOUND"` inside an HTTP 200, and symbols outside your plan as `error: "NOT_ENTITLED"`.

```python
# limit defaults to 10 and pagination fetches every page: always pass limit=250.
for row in client.list_universal_snapshots(
    type="stocks",
    ticker_any_of=["AAPL", "NOPE"],
    limit=250,
):
    if row.error:
        print(row.ticker, "failed:", row.error)
    else:
        print(row.ticker, row.session.price, row.session.previous_close, row.market_status)
```

FinAlly stays on v2 for polling, because v2 is what the code already uses and its `prevDay`, `min` and `lastTrade` cover every need.

### Single Ticker Snapshot

`GET /v2/snapshot/locale/us/markets/stocks/tickers/{stocksTicker}`, the same plans and row shape as the full snapshot.

```python
snap = client.get_snapshot_ticker("stocks", "AAPL")
print(snap.ticker, snap.prev_day.close, snap.updated)
```

### Last Trade and Last Quote

One request per ticker, so these are a poor fit for watchlist polling.

| Endpoint | SDK | Plans |
|---|---|---|
| `GET /v2/last/trade/{ticker}` | `get_last_trade` | Developer (delayed), Advanced, Business + Expansion |
| `GET /v2/last/nbbo/{ticker}` | `get_last_quote` | Advanced (see the plan table in the docs) |

```python
trade = client.get_last_trade("AAPL")
print(trade.price, trade.size, trade.sip_timestamp)  # sip_timestamp is nanoseconds
```

### WebSocket (not used by FinAlly)

- **Hosts:** `wss://socket.massive.com/stocks` for real-time, `wss://delayed.massive.com/stocks` for 15-minute delayed.
- **Channels:** `T.<ticker>` trades, `Q.<ticker>` quotes, `A.<ticker>` per-second bars, `AM.<ticker>` per-minute bars.
- **Limit:** one concurrent connection per asset class by default.

Massive recommends this over polling for continuous data, but `PLAN.md` §3 and §6 chose REST for simplicity, and it would need a long-lived connection plus a dynamic subscription manager. It is the upgrade path if FinAlly ever needs tick-level data.

```python
from massive import WebSocketClient

ws = WebSocketClient(api_key="<key>", subscriptions=["T.AAPL", "T.MSFT"])
ws.run(handle_msg=lambda msgs: [print(m) for m in msgs])  # Blocks
```

## End-of-Day Prices

| Endpoint | Tickers per call | Best for | Plans |
|---|---|---|---|
| Daily Market Summary | **All US stocks** | **Many tickers on a rate-limited plan** | All |
| Previous Day Bar | 1 | One ticker, no date handling | All |
| Daily Ticker Summary | 1 | One date, with pre-market and after-hours | All |
| Custom Bars | 1 | A date range for charts | All |

**When data appears on Stocks Basic:** a day becomes available after midnight US Eastern. Requesting the current day returns 403 at any hour (`{"status":"NOT_AUTHORIZED","message":"Your plan doesn't include this data timeframe..."}`), and retrying sooner doesn't help. Paid plans can get same-day bars.

### Daily Market Summary (FinAlly's end-of-day endpoint)

`GET /v2/aggs/grouped/locale/us/market/stocks/{date}`, with optional `adjusted` (default `true`) and `include_otc` (default `false`). Records go back to 2003-09-10, limited by your plan's history.

```python
bars = client.get_grouped_daily_aggs("2026-11-03")

wanted = {"AAPL", "MSFT", "NVDA"}
closes = {bar.ticker: bar.close for bar in bars if bar.ticker in wanted}
print(closes)
```

```json
{
  "adjusted": true,
  "queryCount": 1,
  "resultsCount": 1,
  "status": "OK",
  "results": [
    {"T": "AAPL", "o": 115.55, "h": 117.59, "l": 114.13, "c": 115.97, "v": 131704427, "vw": 116.3058, "n": 1, "t": 1605042000000}
  ]
}
```

It always returns the whole market, so filter on your side. There is no "latest" option, and the SDK returns `[]` when a date has no results, such as a weekend. Find the latest session by walking back from yesterday:

```python
from datetime import datetime, timedelta
from zoneinfo import ZoneInfo


def latest_session_bars(client, max_days_back=7):
    """Newest non-empty daily summary. Each empty day costs one request."""
    today = datetime.now(ZoneInfo("America/New_York")).date()
    for days_back in range(1, max_days_back + 1):
        session = today - timedelta(days=days_back)
        bars = client.get_grouped_daily_aggs(session.isoformat())
        if bars:
            return session, {bar.ticker: bar for bar in bars}
    return None, {}


session, bars = latest_session_bars(client)
print(session, bars["AAPL"].close)
```

On Stocks Basic, space these calls at least 12 seconds apart to stay within 5 requests/minute.

### Previous Day Bar

`GET /v2/aggs/ticker/{stocksTicker}/prev`. One request per ticker, and it returns a one-element list.

```python
prev = client.get_previous_close_agg("AAPL")[0]
print(prev.ticker, prev.close, prev.timestamp)  # timestamp is milliseconds
```

### Daily Ticker Summary

`GET /v1/open-close/{stocksTicker}/{date}`. The only one here with pre-market and after-hours prices.

```python
day = client.get_daily_open_close_agg("AAPL", "2026-11-03")
print(day.open, day.close, day.pre_market, day.after_hours, day.volume)
```

### Custom Bars

`GET /v2/aggs/ticker/{stocksTicker}/range/{multiplier}/{timespan}/{from}/{to}`. FinAlly doesn't need this today (`PLAN.md` §13 item 14 keeps charts session-only), but it is how to backfill a chart later.

```python
for bar in client.list_aggs("AAPL", 1, "day", "2026-10-01", "2026-10-31", limit=5000):
    print(bar.timestamp, bar.close)  # timestamp is milliseconds
```

`list_aggs` follows `next_url` automatically, so a long range can cost several requests.

## Reference Endpoints

Both are available on all plans.

```python
status = client.get_market_status()                 # GET /v1/marketstatus/now
print(status.market, status.early_hours, status.after_hours)

details = client.get_ticker_details("AAPL")         # GET /v3/reference/tickers/{ticker}
print(details.name, details.active)
```

## Timestamps

| Field | Unit | To Unix seconds |
|---|---|---|
| `lastTrade.t`, `lastQuote.t`, `updated` | Nanoseconds | `/ 1e9` |
| `min.t`, bar `t` | Milliseconds | `/ 1e3` |
| `day`, `prevDay` | Not present in the sample; the SDK leaves `timestamp` as `None` | Use `updated` |

## Errors

The SDK has two separate exception paths, and it does not expose the HTTP status code on either. Confirmed by running the SDK against a local server:

| HTTP | Meaning | SDK raises | What to do |
|---|---|---|---|
| 401 | Missing or invalid key. Live body: `{"status":"ERROR","request_id":"...","error":"API Key was not provided"}` | `massive.exceptions.BadResponse` | Log it; polling will not recover by itself |
| 403 | `NOT_AUTHORIZED`: the plan doesn't cover this endpoint or timeframe | `BadResponse` | Don't retry. For snapshots, fall back to end-of-day |
| 404 | Not found | `BadResponse` | Treat the ticker as unknown |
| 429 | Rate limited: "You've exceeded the maximum requests per minute..." | `urllib3.exceptions.MaxRetryError` | Wait for the next interval |
| 5xx | Server error | `MaxRetryError` | Wait for the next interval |
| n/a | Empty body or no result key | No exception: returns `[]` | Treat as no data |

`BadResponse`'s message is the raw response body, so parse it to read `status`:

```python
import json

from massive.exceptions import BadResponse
from urllib3.exceptions import HTTPError

try:
    snapshots = client.get_snapshot_all("stocks", tickers=["AAPL"])
except BadResponse as e:
    status = json.loads(str(e)).get("status")  # e.g. "NOT_AUTHORIZED"
    print("rejected:", status)
except HTTPError as e:  # MaxRetryError is a subclass
    print("try again later:", e)
```

## SDK Gotchas

Each of these was reproduced with the real SDK.

| Gotcha | Detail | Do this |
|---|---|---|
| **Enum breaks the snapshot URL** | `get_snapshot_all(SnapshotMarketType.STOCKS, ...)` requests `/v2/snapshot/locale/global/markets/SnapshotMarketType.STOCKS/tickers` (2.2.0 and 2.9.1) | Pass the string `"stocks"` |
| **Empty list means everything** | `tickers=[]` sends `?tickers=`, which requests the whole market | Skip the call when the list is empty |
| **Timeouts ignored in 2.2.0** | A stalled server blocked for its full 30 seconds despite `read_timeout=2`. In 2.9.1 the same test raised `MaxRetryError` after 2 seconds | Use 2.9.1 or later, and wrap calls in `asyncio.wait_for` regardless |
| **Retries burn rate limit** | With the default `retries=3`, one 429 or 5xx becomes 4 requests within 0.6 seconds | Use `retries=0` in a poll loop |
| **No `last_trade.timestamp`** | The attribute is `sip_timestamp`, and reading `.timestamp` raises `AttributeError` | Use `sip_timestamp` |
| **`MagicMock` hides all of this** | A mock accepts any attribute or argument | Test with `TickerSnapshot.from_dict(...)` and real JSON |

Between 2.2.0 and 2.9.1, the snapshot and aggregate calls FinAlly uses did not change. The differences that matter here are the timeout fix and a few added fields.

## Polling Recipe

The complete, tested implementation is `MassiveDataSource` in `MARKET_INTERFACE.md`. In short:

```python
import asyncio
import logging

from massive import RESTClient
from massive.exceptions import BadResponse
from urllib3.exceptions import HTTPError


async def poll_forever(api_key, get_tickers, on_price, interval=15.0):
    client = RESTClient(api_key=api_key, retries=0)
    while True:
        tickers = get_tickers()
        if tickers:  # An empty list would request the whole market
            try:
                snapshots = await asyncio.wait_for(
                    asyncio.to_thread(client.get_snapshot_all, "stocks", tickers=tickers),
                    timeout=15,
                )
                for snap in snapshots:
                    on_price(snap)
            except (BadResponse, HTTPError, TimeoutError) as e:
                logging.warning("Poll failed: %s", e)
        await asyncio.sleep(interval)
```

## Defects in the Current Code

`backend/app/market/massive_client.py` cannot produce a price with the locked SDK, even though all 73 tests pass, because its tests use `MagicMock` snapshots. Reproduced with real SDK models:

| # | Location | Defect | Effect |
|---|---|---|---|
| 1 | `_fetch_snapshots` | Passes `SnapshotMarketType.STOCKS`, which builds an invalid URL | Every poll is sent to a path that doesn't exist |
| 2 | `_poll_once` | Reads `snap.last_trade.timestamp`, which doesn't exist | Fixing #1 alone still gives an `AttributeError` per ticker: **the cache stays empty** |
| 3 | `_poll_once` | Divides by 1,000, but the value is in nanoseconds | Once #2 is fixed, timestamps land far in the future |
| 4 | `_poll_once` | Requires `last_trade` | Stocks Starter never gets a price |
| 5 | Class docstring, `PLAN.md` §6 | Assumes free-tier snapshots | Stocks Basic gets a 403 on every poll |

Fixes are in `MARKET_INTERFACE.md`.

### Corrections to `planning/archive/MASSIVE_API.md`

| The archived doc says | Actually |
|---|---|
| Free tier can poll snapshots every 15s | Snapshots are not on the free plan |
| `snap.day.previous_close`, `snap.day.change_percent` | Neither exists. Use `snap.prev_day.close` and `snap.todays_change_percent` |
| `last_trade.timestamp`, in milliseconds | `last_trade.sip_timestamp`, in nanoseconds |
| Passes `SnapshotMarketType.STOCKS` | Pass `"stocks"` |
| JSON keys such as `last_trade`, `open`, `previous_close` | JSON keys are `lastTrade`, `o`, `prevDay.c`; the SDK maps them |
| `quote.bid`, `quote.ask` | `quote.bid_price`, `quote.ask_price` |
| 5xx errors are simply retried by the client | They are, but so are 429s, and both end in `MaxRetryError` |
| "Reads `MASSIVE_API_KEY` automatically" | True, but read once at import time, so pass it explicitly |

## Live Verification

Run this once with a real key before relying on Massive mode. It uses one request per step, so on Stocks Basic stay under 5 in a minute.

```python
"""Usage: MASSIVE_API_KEY=... uv run python smoke.py"""
import json
import os
from datetime import datetime, timedelta
from zoneinfo import ZoneInfo

from massive import RESTClient
from massive.exceptions import BadResponse

client = RESTClient(api_key=os.environ["MASSIVE_API_KEY"], retries=0)

try:
    rows = client.get_snapshot_all("stocks", tickers=["AAPL", "ZZZZNOTREAL"])
    print("Tickers returned:", [row.ticker for row in rows])
    for row in rows:
        print(
            row.ticker,
            "trade:", row.last_trade.price if row.last_trade else None,
            "minute:", row.min.close if row.min else None,
            "day:", row.day.close if row.day else None,
            "prev:", row.prev_day.close if row.prev_day else None,
        )
except BadResponse as e:
    print("Snapshot rejected:", json.loads(str(e)).get("status"))

yesterday = datetime.now(ZoneInfo("America/New_York")).date() - timedelta(days=1)
print("Rows for", yesterday, "=", len(client.get_grouped_daily_aggs(yesterday.isoformat())))

try:
    print("Single unknown ticker:", client.get_snapshot_ticker("stocks", "ZZZZNOTREAL"))
except BadResponse as e:
    print("Single unknown ticker rejected:", e)
```

What it settles, none of which the docs state:

- Does the v2 snapshot omit unknown tickers? (The design tolerates either.)
- What does the single-ticker snapshot return for an unknown symbol? (The design assumes a `BadResponse` or an empty result.)
- Does Stocks Starter really return `min` outside market hours? (If not, the `day` and `prevDay` fallbacks apply.)
- Does a 403 on the snapshot carry `status: "NOT_AUTHORIZED"`? The knowledge base says so; the automatic fallback to end-of-day depends on it.

## Sources

- [Full Market Snapshot](https://massive.com/docs/rest/stocks/snapshots/full-market-snapshot)
- [Unified Snapshot](https://massive.com/docs/rest/stocks/snapshots/unified-snapshot)
- [Single Ticker Snapshot](https://massive.com/docs/rest/stocks/snapshots/single-ticker-snapshot)
- [Last Trade](https://massive.com/docs/rest/stocks/trades-quotes/last-trade)
- [Daily Market Summary](https://massive.com/docs/rest/stocks/aggregates/daily-market-summary)
- [Previous Day Bar](https://massive.com/docs/rest/stocks/aggregates/previous-day-bar)
- [Daily Ticker Summary](https://massive.com/docs/rest/stocks/aggregates/daily-ticker-summary)
- [Custom Bars](https://massive.com/docs/rest/stocks/aggregates/custom-bars)
- [Market Status](https://massive.com/docs/rest/stocks/market-operations/market-status)
- [Ticker Overview](https://massive.com/docs/rest/stocks/tickers/ticker-overview)
- [REST Quickstart](https://massive.com/docs/rest/quickstart)
- [WebSocket Quickstart](https://massive.com/docs/websocket/quickstart) and [Stocks overview](https://massive.com/docs/websocket/stocks/overview)
- [Request limits](https://massive.com/knowledge-base/article/what-is-the-request-limit-for-massives-restful-apis)
- [Basic plan data availability](https://massive.com/knowledge-base/article/on-the-free-basic-plan-when-can-i-pull-the-previous-trading-days-data)
- [Latest data in one request](https://massive.com/knowledge-base/article/does-massive-have-an-endpoint-that-returns-the-latest-trades-quotes-and-aggregates-in-one-request)
- [Maximum tickers per request](https://massive.com/knowledge-base/article/what-is-the-max-number-of-tickers-i-can-pass-through-polygons-snapshot)
- [Pricing](https://massive.com/pricing) and [Massive vs yfinance](https://massive.com/compare/massive-vs-yfinance) (vendor-written)
- [Python SDK](https://github.com/massive-com/client-python)
