# Market Data Interface — Unified Python API

This document describes the unified Python interface for retrieving stock prices in FinAlly. The design ensures that all downstream code (SSE streaming, portfolio valuation, trade execution) is completely agnostic to whether prices come from the GBM simulator or the Massive REST API.

---

## Architecture Overview

```
                   ┌─────────────────────────────────┐
                   │         MarketDataSource         │
                   │         (Abstract Base)          │
                   └─────────────┬───────────────────┘
                                 │ implements
               ┌─────────────────┴──────────────────┐
               │                                     │
   ┌───────────┴────────────┐       ┌───────────────┴─────────────┐
   │   SimulatorDataSource  │       │      MassiveDataSource      │
   │  (GBM — default)       │       │  (REST poller — if API key) │
   └───────────┬────────────┘       └───────────────┬─────────────┘
               │                                     │
               └──────────────┬──────────────────────┘
                              │ writes to
                    ┌─────────▼──────────┐
                    │     PriceCache     │
                    │  (single source    │
                    │   of truth)        │
                    └─────┬──────────────┘
                          │ reads from
           ┌──────────────┼──────────────────────┐
           │              │                      │
    ┌──────▼──────┐  ┌────▼────────┐  ┌──────────▼──────┐
    │  SSE stream │  │  Portfolio  │  │ Trade execution  │
    │ /api/stream │  │ valuation   │  │  (buy/sell)      │
    └─────────────┘  └─────────────┘  └──────────────────┘
```

**Key invariant:** Producers (simulator or Massive poller) write to the cache on their own schedule. Consumers never call the data source directly — they always read from `PriceCache`.

---

## Module Layout

All market data code lives in `backend/app/market/`.

| File | Class/Function | Role |
|------|----------------|------|
| `models.py` | `PriceUpdate` | Immutable data record for a single price tick |
| `cache.py` | `PriceCache` | Thread-safe in-memory store; single source of truth |
| `interface.py` | `MarketDataSource` | Abstract base class (ABC) defining the producer contract |
| `simulator.py` | `GBMSimulator`, `SimulatorDataSource` | GBM-based price simulation |
| `massive_client.py` | `MassiveDataSource` | Massive REST API polling client |
| `factory.py` | `create_market_data_source()` | Selects simulator vs. Massive based on env var |
| `seed_prices.py` | constants | Seed prices, GBM parameters, correlation groups |
| `stream.py` | `create_stream_router()` | FastAPI SSE endpoint factory |

Public re-exports from `app/market/__init__.py`:
```python
from app.market import (
    PriceCache,
    PriceUpdate,
    MarketDataSource,
    create_market_data_source,
    create_stream_router,
)
```

---

## Core Types

### `PriceUpdate`

Immutable frozen dataclass representing one price tick.

```python
@dataclass(frozen=True, slots=True)
class PriceUpdate:
    ticker: str
    price: float            # Current price (rounded to 2 decimal places)
    previous_price: float   # Price from the previous update
    timestamp: float        # Unix seconds (time.time())

    # Computed properties (no stored state):
    @property
    def change(self) -> float: ...          # price - previous_price
    @property
    def change_percent(self) -> float: ...  # % change
    @property
    def direction(self) -> str: ...         # "up" | "down" | "flat"

    def to_dict(self) -> dict: ...
    # Returns: {ticker, price, previous_price, timestamp, change, change_percent, direction}
```

`to_dict()` is the serialization format used for SSE events and API responses.

---

### `PriceCache`

Thread-safe in-memory store. One instance shared across the entire application lifetime.

```python
cache = PriceCache()

# Write (called by the data source only)
update: PriceUpdate = cache.update("AAPL", price=190.42)
cache.update("AAPL", price=190.55, timestamp=1700000000.0)  # optional explicit ts

# Read (called by consumers)
update: PriceUpdate | None = cache.get("AAPL")
price: float | None = cache.get_price("AAPL")        # shorthand for update.price
all_prices: dict[str, PriceUpdate] = cache.get_all() # shallow copy — safe to iterate

# Removal (called when a ticker leaves the watchlist)
cache.remove("AAPL")

# SSE change detection
version: int = cache.version  # monotonically increases on every update()
```

**First-update behavior:** When a ticker is first seen, `previous_price == price` so `direction == "flat"`. The second update produces a real direction.

**Thread safety:** All methods acquire a `threading.Lock`. The SSE stream (asyncio) and background poller (asyncio thread) read concurrently without races.

---

### `MarketDataSource` (Abstract Base Class)

```python
class MarketDataSource(ABC):
    @abstractmethod
    async def start(self, tickers: list[str]) -> None:
        """Begin producing price updates. Call exactly once at startup."""

    @abstractmethod
    async def stop(self) -> None:
        """Cancel background task. Safe to call multiple times."""

    @abstractmethod
    async def add_ticker(self, ticker: str) -> None:
        """Include a new ticker in future updates. No-op if already present."""

    @abstractmethod
    async def remove_ticker(self, ticker: str) -> None:
        """Drop a ticker. Also removes it from PriceCache."""

    @abstractmethod
    def get_tickers(self) -> list[str]:
        """Return the currently tracked tickers."""
```

---

## Factory — Selecting a Data Source

```python
from app.market import PriceCache, create_market_data_source

cache = PriceCache()
source = create_market_data_source(cache)
```

`create_market_data_source` reads `MASSIVE_API_KEY` from the environment:

- **Key present and non-empty** → returns `MassiveDataSource(api_key=..., price_cache=cache)`
- **Key absent or empty** → returns `SimulatorDataSource(price_cache=cache)`

The caller never needs to import `MassiveDataSource` or `SimulatorDataSource` directly.

---

## Lifecycle (FastAPI Integration)

The recommended integration is via FastAPI's `lifespan` context manager, which runs startup/shutdown logic cleanly:

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI
from app.market import PriceCache, create_market_data_source, create_stream_router

DEFAULT_TICKERS = ["AAPL", "GOOGL", "MSFT", "AMZN", "TSLA", "NVDA", "META", "JPM", "V", "NFLX"]

price_cache = PriceCache()
market_source = create_market_data_source(price_cache)

@asynccontextmanager
async def lifespan(app: FastAPI):
    await market_source.start(DEFAULT_TICKERS)
    yield
    await market_source.stop()

app = FastAPI(lifespan=lifespan)
app.include_router(create_stream_router(price_cache))
```

---

## Dynamic Watchlist Management

When the user adds or removes a ticker (via REST API or AI chat), the backend calls the data source directly. The cache is updated in the same call:

```python
# Adding a ticker (watchlist POST handler)
await market_source.add_ticker("PYPL")
# Cache immediately has a seed price; SSE stream will emit it on the next tick

# Removing a ticker (watchlist DELETE handler)
await market_source.remove_ticker("NFLX")
# Cache entry is deleted; SSE stream will stop emitting it
```

**Simulator behavior for unknown tickers:** If the ticker is not in `seed_prices.py`, the simulator assigns a random starting price in the $50–$300 range and uses the `DEFAULT_PARAMS` volatility/drift. See `MARKET_SIMULATOR.md`.

**Massive behavior for unknown tickers:** The next poll will include the new ticker in the `tickers` query parameter. If Massive doesn't recognise it (invalid symbol), the snapshot for that ticker is absent from the response; the cache entry stays absent until a valid price arrives.

---

## Reading Prices in API Handlers

Downstream handlers should always read from the cache, never from the data source:

```python
from app.market import PriceCache

# Injected or module-level singleton
price_cache: PriceCache = ...

# Single ticker price
price = price_cache.get_price("AAPL")  # float | None

# Full snapshot for portfolio valuation
positions = get_all_positions(user_id="default")
all_prices = price_cache.get_all()  # dict[str, PriceUpdate]

for pos in positions:
    update = all_prices.get(pos.ticker)
    if update:
        current_value = pos.quantity * update.price
        unrealized_pnl = current_value - (pos.quantity * pos.avg_cost)
```

---

## SSE Streaming Endpoint

`create_stream_router(price_cache)` returns a FastAPI `APIRouter` with one endpoint:

```
GET /api/stream/prices   (Content-Type: text/event-stream)
```

The endpoint uses version-based change detection: it samples `cache.version` each loop iteration and only emits a new SSE event when the version has changed. This avoids sending redundant events when no prices have moved.

**SSE event format:**
```
data: {"ticker":"AAPL","price":190.42,"previous_price":190.38,"timestamp":1700000000.12,"change":0.04,"change_percent":0.021,"direction":"up"}

data: {"ticker":"TSLA","price":248.10,"previous_price":250.00,"timestamp":1700000000.12,"change":-1.9,"change_percent":-0.76,"direction":"down"}
```

Each SSE message is one `PriceUpdate.to_dict()` payload. The client accumulates these with `EventSource` and the browser handles automatic reconnection.

---

## PriceUpdate SSE Payload Reference

| Field | Type | Description |
|-------|------|-------------|
| `ticker` | string | Ticker symbol, e.g. `"AAPL"` |
| `price` | float | Current price (2 decimal places) |
| `previous_price` | float | Price from previous update |
| `timestamp` | float | Unix seconds |
| `change` | float | `price - previous_price` |
| `change_percent` | float | `(change / previous_price) * 100` |
| `direction` | string | `"up"` \| `"down"` \| `"flat"` |

---

## Adding a New Data Source

To add a third data source (e.g., Alpha Vantage, Alpaca):

1. Create `backend/app/market/alpaca_client.py`
2. Define a class that extends `MarketDataSource` and implements all five abstract methods
3. Update `factory.py` to check a new env var (e.g., `ALPACA_API_KEY`) and return the new class
4. All downstream code continues to work unchanged — it only reads from `PriceCache`
