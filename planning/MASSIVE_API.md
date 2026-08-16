# Massive API — Reference Documentation

Massive (formerly Polygon.io, rebranded October 2025) provides real-time and historical market data for US stocks, options, indices, forex, cryptocurrencies, and futures via REST endpoints and WebSocket streams under `api.polygon.io` (unchanged from the rebrand).

## Authentication

All requests require an API key. Pass it in the `Authorization` header or via the Python client constructor:

```python
from massive import RESTClient

# Explicit key
client = RESTClient(api_key="YOUR_KEY")

# Or read from environment variable MASSIVE_API_KEY (preferred)
client = RESTClient()
```

Store the key in the `.env` file as `MASSIVE_API_KEY=...`. Never hardcode it in source files.

Sign up at: https://massive.com/dashboard/signup  
Retrieve keys at: https://massive.com/dashboard/keys

---

## Subscription Plans & Rate Limits

| Plan | Price | Rate Limit | Data Recency |
|------|-------|------------|--------------|
| **Basic** | Free | 5 calls/min | End-of-day only |
| **Starter** | $29/month | Unlimited | 15-min delayed |
| **Developer** | $79/month | Unlimited | 15-min delayed |
| **Advanced** | $199/month | Unlimited | Real-time |
| **Business** | Custom | Unlimited | Real-time + FMV |

**Implication for FinAlly:** With the Basic (free) plan, the maximum safe poll rate is one call every 15 seconds. With any paid plan (Starter+), polling every 2–15 seconds is viable.

---

## Installation

```bash
pip install -U massive
```

Requires Python 3.9+.

---

## Key Endpoints for FinAlly

### 1. Full Market Snapshot — Multiple Tickers (Primary)

Retrieves a snapshot (current minute bar, daily bar, previous day bar, last trade, last quote) for a comma-separated list of tickers in one API call.

**HTTP method / path:**
```
GET /v2/snapshot/locale/us/markets/stocks/tickers?tickers=AAPL,TSLA,GOOG
```

**Python client:**
```python
from massive import RESTClient
from massive.rest.models import SnapshotMarketType

client = RESTClient()

snapshots = client.get_snapshot_all(
    market_type=SnapshotMarketType.STOCKS,
    tickers=["AAPL", "TSLA", "GOOG", "MSFT"],
)

for snap in snapshots:
    print(snap.ticker)
    print(snap.last_trade.price)           # float — last traded price
    print(snap.last_trade.timestamp)       # int — Unix milliseconds
    print(snap.day.close)                  # float — today's close so far
    print(snap.todays_change_perc)         # float — % change vs previous close
    print(snap.prev_day.close)             # float — previous day's close
```

**Response shape** (abbreviated):

```json
{
  "status": "OK",
  "count": 2,
  "tickers": [
    {
      "ticker": "AAPL",
      "todaysChange": 0.98,
      "todaysChangePerc": 0.82,
      "updated": 1605195918306274000,
      "day": {
        "o": 119.62, "h": 120.53, "l": 118.81, "c": 120.42, "v": 28727868, "vw": 119.725
      },
      "min": {
        "o": 120.435, "h": 120.468, "l": 120.37, "c": 120.4201, "v": 270796,
        "vw": 120.4129, "t": 1684428720000, "n": 762
      },
      "prevDay": {
        "o": 117.19, "h": 119.63, "l": 116.44, "c": 119.49, "v": 110597265
      },
      "lastTrade": {
        "p": 120.47,
        "s": 236,
        "t": 1605195918306274000,
        "x": 10
      },
      "lastQuote": {
        "P": 120.47,
        "p": 120.46,
        "S": 4,
        "s": 8,
        "t": 1605195918507251700
      }
    }
  ]
}
```

**Field reference for `lastTrade`:**

| Field | Meaning |
|-------|---------|
| `p` | Price |
| `s` | Size (shares) |
| `t` | Timestamp (Unix nanoseconds) |
| `x` | Exchange ID |

**Field reference for `day` / `min` / `prevDay`:**

| Field | Meaning |
|-------|---------|
| `o` | Open |
| `h` | High |
| `l` | Low |
| `c` | Close |
| `v` | Volume |
| `vw` | Volume-weighted average price (VWAP) |
| `t` | Start timestamp of bar (min only) |
| `n` | Number of transactions (min only) |

**Important:** Timestamps in `lastTrade` are Unix **nanoseconds**; in `min.t` they are Unix **milliseconds**. Divide nanosecond timestamps by `1e9` and millisecond timestamps by `1e3` to get Unix seconds.

---

### 2. Single Ticker Snapshot

Returns the same data as the full market snapshot but for one ticker. Useful for resolving a ticker not in the current batch.

**HTTP method / path:**
```
GET /v2/snapshot/locale/us/markets/stocks/tickers/{stocksTicker}
```

**Python client:**
```python
snap = client.get_snapshot("stocks", "AAPL")
price = snap.last_trade.price
```

**Example response:**
```json
{
  "request_id": "657e430f1ae768891f018e08e03598d8",
  "status": "OK",
  "ticker": {
    "ticker": "AAPL",
    "todaysChange": 0.98,
    "todaysChangePerc": 0.82,
    "updated": 1605195918306274000,
    "day": {"c": 120.4229, "h": 120.53, "l": 118.81, "o": 119.62, "v": 28727868, "vw": 119.725},
    "min": {"c": 120.4201, "h": 120.468, "l": 120.37, "o": 120.435, "v": 270796, "vw": 120.4129, "t": 1684428720000, "n": 762},
    "prevDay": {"c": 119.49, "h": 119.63, "l": 116.44, "o": 117.19, "v": 110597265},
    "lastTrade": {"p": 120.47, "s": 236, "t": 1605195918306274000, "x": 10},
    "lastQuote": {"P": 120.47, "p": 120.46, "S": 4, "s": 8, "t": 1605195918507251700},
    "fmv": null
  }
}
```

---

### 3. Unified Snapshot — Multi-Asset (Alternative)

Returns snapshots across multiple asset classes in one call, with up to 250 tickers per request via `ticker.any_of`.

**HTTP method / path:**
```
GET /v3/snapshot?ticker.any_of=AAPL,TSLA,GOOG&limit=250
```

**Python client:**
```python
results = client.list_universal_snapshots(
    params={"ticker.any_of": "AAPL,TSLA,GOOG", "limit": 250}
)
for r in results:
    print(r.ticker, r.session.close, r.last_trade.price)
```

The `session` object contains `open`, `close`, `high`, `low`, `volume`, `change`, `change_percent` for the current day.

---

### 4. Last Trade

Returns the most recent trade for a single ticker. Lower overhead than a full snapshot when only the price matters.

**Python client:**
```python
trade = client.get_last_trade("AAPL")
print(trade.price)      # float
print(trade.size)       # int — shares traded
print(trade.timestamp)  # int — Unix nanoseconds
```

---

### 5. Previous Day Bar (OHLC)

Useful for computing daily change % when building a "today's change" column.

**HTTP method / path:**
```
GET /v2/aggs/ticker/{ticker}/prev
```

**Python client:**
```python
prev = client.get_previous_close("AAPL")
print(prev.close)   # yesterday's closing price
print(prev.open)    # yesterday's open
print(prev.high)    # yesterday's high
print(prev.low)     # yesterday's low
print(prev.volume)  # yesterday's volume
```

---

### 6. Historical Aggregate Bars (OHLCV)

Returns a sequence of OHLCV bars for a ticker over a date range. Useful for populating the main chart on first load.

```python
aggs = []
for bar in client.list_aggs(
    ticker="AAPL",
    multiplier=1,
    timespan="minute",
    from_="2024-01-01",
    to="2024-01-31",
    limit=50000,
):
    aggs.append(bar)
    # bar.open, bar.high, bar.low, bar.close, bar.volume, bar.timestamp (ms)
```

---

## Market Status

Check whether the market is currently open or closed (affects whether snapshot prices are stale):

```python
status = client.get_market_status()
print(status.market)    # "open" | "closed" | "extended-hours"
```

---

## Error Handling

The Massive client raises exceptions on HTTP errors. Common codes:

| HTTP Status | Cause | Action |
|-------------|-------|--------|
| `401 Unauthorized` | Invalid or missing API key | Check `MASSIVE_API_KEY` env var |
| `403 Forbidden` | Endpoint not in your plan | Upgrade plan or use different endpoint |
| `429 Too Many Requests` | Rate limit exceeded | Back off; lower poll frequency |
| `404 Not Found` | Unknown ticker | Remove from watchlist or return error |
| `503 Service Unavailable` | Massive API down | Retry with exponential backoff |

Catch and log, then continue — transient failures should not crash the polling loop:

```python
try:
    snapshots = client.get_snapshot_all(
        market_type=SnapshotMarketType.STOCKS,
        tickers=tickers,
    )
except Exception as e:
    logger.error("Massive poll failed: %s", e)
    # Loop will retry on next interval
```

---

## Data Freshness Notes

- Snapshot data is cleared at 3:30 AM EST and begins updating as exchanges open (~4:00 AM EST).
- On Starter/Developer plans, all snapshot data is 15 minutes delayed — not suitable for live trading simulation during market hours.
- On Advanced/Business plans, data is real-time during market hours and includes extended-hours quotes.
- Outside US market hours (9:30 AM – 4:00 PM ET), `lastTrade.p` reflects the most recent trade from that day's session.

---

## How FinAlly Uses the Massive API

FinAlly polls `GET /v2/snapshot/locale/us/markets/stocks/tickers` with the user's entire watchlist as a comma-separated `tickers` query parameter. This batches all tickers into one API call per poll cycle, minimizing rate-limit pressure on free-tier accounts.

The poller runs in a background `asyncio` task, calling the synchronous REST client in a thread via `asyncio.to_thread()` to avoid blocking the event loop. Extracted prices are written to the shared `PriceCache`. See `MARKET_INTERFACE.md` for the full architecture.
