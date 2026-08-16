# Market Simulator — Design & Code Structure

The simulator is the default market data source for FinAlly. It runs entirely in-process with no external dependencies and generates realistic, visually compelling price movement using Geometric Brownian Motion (GBM) with correlated ticker moves.

---

## Why Simulate?

- Zero cost — no API key required for development or demo
- Instant price updates every 500ms — no polling delay
- Controllable behavior — deterministic seeds, tunable volatility
- Arbitrary tickers — any symbol can be added without API validation
- Visually dramatic — correlated sector moves and random shocks make it feel real

---

## Mathematical Foundation

### Geometric Brownian Motion (GBM)

GBM is the standard model for stock price simulation. Each tick advances the price by:

```
S(t + dt) = S(t) × exp((μ - σ²/2) × dt + σ × √dt × Z)
```

Where:

| Symbol | Meaning |
|--------|---------|
| `S(t)` | Current price |
| `μ` (mu) | Annualized drift — the expected return per year |
| `σ` (sigma) | Annualized volatility — standard deviation of log returns per year |
| `dt` | Time step as a fraction of a trading year |
| `Z` | Standard normal random variable N(0,1) |

The exponent form ensures prices stay strictly positive (no negative prices) and returns are log-normally distributed, consistent with observed equity price behavior.

**In code:**
```python
drift = (mu - 0.5 * sigma**2) * dt
diffusion = sigma * math.sqrt(dt) * Z
S_new = S_old * math.exp(drift + diffusion)
```

### Time Step Size

Prices update every 500ms. The time step is expressed as a fraction of a trading year:

```
Trading seconds per year = 252 days × 6.5 hours/day × 3600 sec/hour = 5,896,800
dt = 0.5 / 5,896,800 ≈ 8.48 × 10⁻⁸
```

This tiny dt produces sub-cent moves per tick that accumulate naturally and realistically over minutes and hours. With σ=0.25 (25% annual vol) and dt=8.48e-8:

```
Per-tick standard deviation = σ × √dt ≈ 0.25 × 0.000291 ≈ 0.0073%
```

Over 120 ticks (one minute), this compounds to roughly 0.08% price drift — realistic intraday noise.

---

## Correlated Moves

Real stocks don't move independently — tech stocks tend to rise and fall together. The simulator models this with a **Cholesky decomposition** of a correlation matrix.

### Correlation Structure

| Pair | Correlation (ρ) |
|------|-----------------|
| Two tech stocks (AAPL/GOOGL/MSFT/AMZN/META/NVDA/NFLX) | 0.60 |
| Two finance stocks (JPM/V) | 0.50 |
| TSLA with anything | 0.30 (acts independently) |
| Cross-sector (tech ↔ finance) | 0.30 |
| Unknown ticker with anything | 0.30 |

### How Cholesky Works

Given n tickers, build an n×n correlation matrix Σ where `Σ[i,j]` is the pairwise correlation. Then compute the Cholesky factor L such that `Σ = L × Lᵀ`.

To generate correlated normal draws:
1. Draw n independent standard normals: `z_independent ~ N(0,1)ⁿ`
2. Apply Cholesky: `z_correlated = L @ z_independent`
3. `z_correlated[i]` has the right covariance structure — ticker i's price move is correlated with ticker j's at coefficient `Σ[i,j]`

**In code:**
```python
import numpy as np

z_independent = np.random.standard_normal(n)
z_correlated = self._cholesky @ z_independent  # shape (n,)

# Then for each ticker i:
diffusion = sigma_i * math.sqrt(dt) * z_correlated[i]
```

The Cholesky matrix is rebuilt whenever tickers are added or removed (O(n²), but n < 50 in practice).

---

## Random Shock Events

To make the simulation visually dramatic, each tick has a small probability of a sudden large price move:

```python
# ~0.1% chance per tick per ticker
if random.random() < 0.001:
    shock_magnitude = random.uniform(0.02, 0.05)   # 2–5% move
    shock_sign = random.choice([-1, 1])
    price *= 1 + shock_magnitude * shock_sign
```

With 10 tickers updating at 2 ticks/second: expected ~1 shock event every 50 seconds. This gives the terminal a "breaking news" feel without being disruptive.

---

## Code Structure

### `GBMSimulator` — Pure Price Engine

`GBMSimulator` is a synchronous, framework-agnostic class. It holds price state and advances prices when `step()` is called. It knows nothing about asyncio, FastAPI, or the cache.

```python
class GBMSimulator:
    def __init__(self, tickers: list[str], dt: float = DEFAULT_DT, event_probability: float = 0.001): ...

    def step(self) -> dict[str, float]:
        """Advance all tickers one time step. Returns {ticker: new_price}."""

    def add_ticker(self, ticker: str) -> None:
        """Add a ticker. Rebuilds Cholesky. New tickers get seed price or random $50–$300."""

    def remove_ticker(self, ticker: str) -> None:
        """Remove a ticker. Rebuilds Cholesky."""

    def get_price(self, ticker: str) -> float | None:
        """Current price for a single ticker."""

    def get_tickers(self) -> list[str]:
        """Current list of tracked tickers."""
```

Internal state:
- `_tickers: list[str]` — ordered list (order matters for Cholesky indexing)
- `_prices: dict[str, float]` — current price per ticker
- `_params: dict[str, dict]` — `{mu, sigma}` per ticker
- `_cholesky: np.ndarray | None` — Cholesky factor of correlation matrix

### `SimulatorDataSource` — asyncio Adapter

`SimulatorDataSource` wraps `GBMSimulator` to implement the `MarketDataSource` ABC. It runs an asyncio background task that calls `simulator.step()` every 500ms and writes results to `PriceCache`.

```python
class SimulatorDataSource(MarketDataSource):
    def __init__(self, price_cache: PriceCache, update_interval: float = 0.5, event_probability: float = 0.001): ...

    async def start(self, tickers: list[str]) -> None:
        """Create GBMSimulator, seed cache with initial prices, start background task."""

    async def stop(self) -> None:
        """Cancel the background task. Safe to call multiple times."""

    async def add_ticker(self, ticker: str) -> None:
        """Delegate to GBMSimulator.add_ticker(); immediately seed cache with starting price."""

    async def remove_ticker(self, ticker: str) -> None:
        """Delegate to GBMSimulator.remove_ticker(); remove from cache."""

    def get_tickers(self) -> list[str]:
        """Delegate to GBMSimulator.get_tickers()."""

    async def _run_loop(self) -> None:
        """Core loop: call step(), write prices to cache, sleep."""
```

**Startup sequence:**
1. Instantiate `GBMSimulator(tickers)` — seeds initial prices from `SEED_PRICES`
2. Immediately write all initial prices to `PriceCache` (cache is non-empty before first tick)
3. Launch `asyncio.create_task(_run_loop())` — named `"simulator-loop"` for debugging

**Loop:**
```python
async def _run_loop(self) -> None:
    while True:
        try:
            prices = self._sim.step()
            for ticker, price in prices.items():
                self._cache.update(ticker=ticker, price=price)
        except Exception:
            logger.exception("Simulator step failed")
        await asyncio.sleep(self._interval)
```

Exceptions are caught-and-logged rather than re-raised so a bad numpy draw doesn't crash the whole server.

---

## Seed Prices and Parameters

Defined in `backend/app/market/seed_prices.py`:

```python
SEED_PRICES = {
    "AAPL": 190.00,
    "GOOGL": 175.00,
    "MSFT": 420.00,
    "AMZN": 185.00,
    "TSLA": 250.00,
    "NVDA": 800.00,
    "META": 500.00,
    "JPM":  195.00,
    "V":    280.00,
    "NFLX": 600.00,
}

TICKER_PARAMS = {
    "AAPL":  {"sigma": 0.22, "mu": 0.05},
    "GOOGL": {"sigma": 0.25, "mu": 0.05},
    "MSFT":  {"sigma": 0.20, "mu": 0.05},
    "AMZN":  {"sigma": 0.28, "mu": 0.05},
    "TSLA":  {"sigma": 0.50, "mu": 0.03},  # High vol, low drift
    "NVDA":  {"sigma": 0.40, "mu": 0.08},  # High vol, high drift
    "META":  {"sigma": 0.30, "mu": 0.05},
    "JPM":   {"sigma": 0.18, "mu": 0.04},  # Low vol (bank)
    "V":     {"sigma": 0.17, "mu": 0.04},  # Low vol (payments)
    "NFLX":  {"sigma": 0.35, "mu": 0.05},
}

# Used for any ticker not in TICKER_PARAMS
DEFAULT_PARAMS = {"sigma": 0.25, "mu": 0.05}
```

**Sigma choices** reflect approximate real-world annualized volatility for these names. TSLA at 50% is realistic (it has historically ranged 40–80%). NVDA at 40% is conservative. Banks at 17–18% reflect their lower price sensitivity.

**Mu** is 5% annualized expected return for most names — slight upward drift, enough to be visible over longer sessions but not so large it dominates the volatility.

---

## Handling Unknown Tickers

When the user or AI adds a ticker not in `SEED_PRICES`:

```python
def _add_ticker_internal(self, ticker: str) -> None:
    self._tickers.append(ticker)
    self._prices[ticker] = SEED_PRICES.get(ticker, random.uniform(50.0, 300.0))
    self._params[ticker] = TICKER_PARAMS.get(ticker, dict(DEFAULT_PARAMS))
```

- **Starting price:** random in [$50, $300] — wide enough to cover most US equities
- **Volatility:** 25% annualized — slightly above market average, gives visible movement
- **Correlation:** `CROSS_GROUP_CORR = 0.3` with everything else (treated as cross-sector unknown)

This means the simulator accepts any ticker string the user types and synthesizes plausible movement. There is no validation that the ticker is a real stock — that responsibility belongs to the Massive client when `MASSIVE_API_KEY` is set.

---

## Performance Characteristics

| Metric | Value |
|--------|-------|
| Update interval | 500ms |
| CPU per tick (10 tickers) | < 1ms (numpy GBM + cache write) |
| Memory (10 tickers) | ~5 KB (correlation matrix + prices) |
| Cholesky rebuild cost | O(n²) per add/remove, negligible for n < 50 |
| Shock event frequency | ~1 per 50 seconds (10 tickers, 2 ticks/sec) |

---

## Testing the Simulator

The test suite in `backend/tests/market/` covers:

- `test_simulator.py` — 17 tests: GBM math correctness, Cholesky correlation, shock events, add/remove tickers, edge cases (empty ticker list, single ticker)
- `test_simulator_source.py` — 10 integration tests: `SimulatorDataSource` lifecycle, cache population, add/remove during running simulation

Run with:
```bash
cd backend
uv run --extra dev pytest tests/market/test_simulator.py tests/market/test_simulator_source.py -v
```

---

## Demo

A Rich terminal dashboard demonstrates the simulator live:

```bash
cd backend
uv run market_data_demo.py
```

Shows all 10 tickers with live-updating prices, sparklines, color-coded direction arrows, and an event log for shock moves. Runs for 60 seconds or until Ctrl+C.
