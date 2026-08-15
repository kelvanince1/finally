# FinAlly — AI Trading Workstation

An AI-powered trading workstation that streams live market data, lets you trade a simulated portfolio, and integrates an LLM assistant that can analyze positions and execute trades through natural language.

Built as a capstone project for an agentic AI coding course — the entire application is written by orchestrated AI coding agents.

## What It Does

- **Live price streaming** via SSE — prices flash green/red on uptick/downtick with sparkline mini-charts
- **Simulated portfolio** — start with $10,000 virtual cash, buy/sell at market price, instant fill
- **Portfolio visualizations** — treemap heatmap sized by position weight, P&L chart over time
- **AI chat assistant** — ask about your portfolio, get analysis, and have the AI execute trades or manage your watchlist automatically

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | Next.js (TypeScript, static export) |
| Backend | FastAPI (Python, managed with `uv`) |
| Database | SQLite (lazy-initialized, volume-mounted) |
| Real-time | Server-Sent Events (SSE) |
| AI | LiteLLM → OpenRouter → Cerebras inference |
| Market data | GBM simulator (default) or Polygon.io REST API |
| Deployment | Single Docker container on port 8000 |

## Quick Start

```bash
cp .env.example .env
# Add your OPENROUTER_API_KEY to .env

./scripts/start_mac.sh       # macOS/Linux
# or
.\scripts\start_windows.ps1  # Windows PowerShell
```

Open [http://localhost:8000](http://localhost:8000).

To stop:

```bash
./scripts/stop_mac.sh
```

## Environment Variables

```bash
# Required — LLM chat functionality
OPENROUTER_API_KEY=your-key-here

# Optional — real market data via Polygon.io; simulator used if absent
MASSIVE_API_KEY=

# Optional — deterministic mock LLM responses for testing
LLM_MOCK=false
```

## Market Data

By default the app uses a built-in **GBM simulator** — no API key required. Prices update every ~500ms with correlated sector moves and occasional random shock events for realism.

Set `MASSIVE_API_KEY` to switch to live Polygon.io data.

## Development

The backend market data subsystem (`backend/app/market/`) is complete with 73 passing tests and 84% coverage. A terminal demo is available:

```bash
cd backend
uv run market_data_demo.py
```

Run backend tests:

```bash
cd backend
uv run pytest
```

## Project Status

| Component | Status |
|---|---|
| Market data backend | Complete |
| REST API + database | In progress |
| Frontend UI | In progress |
| AI chat integration | In progress |
| Docker build | In progress |
| E2E tests | In progress |

## License

MIT
