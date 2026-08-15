# Review of planning/PLAN.md

## Findings

1. **High - API contracts are underspecified for parallel agent work.**  
   `planning/PLAN.md:249` lists endpoint paths and short descriptions, but it does not define request and response schemas for `/api/portfolio`, `/api/watchlist`, `/api/portfolio/trade`, `/api/portfolio/history`, or `/api/chat`. Because this project is explicitly split across agents, the frontend and backend are likely to invent incompatible shapes for portfolio totals, position rows, validation errors, trade results, chat actions, and timestamps. Add a small contract section with JSON examples, status codes, and error bodies before implementation continues.

2. **High - Docker volume documentation contradicts itself.**  
   `planning/PLAN.md:399` uses a named Docker volume (`finally-data:/app/db`), but `planning/PLAN.md:402` says the project-root `db/` directory maps to `/app/db`. Those are different persistence models. If agents implement scripts or Docker docs from different lines, data may persist in an unexpected place or the repo `db/` folder may never be used. Pick one: either a named volume or a bind mount like `$(pwd)/db:/app/db`, then align the Docker command, scripts, and directory-structure notes.

3. **High - Chat persistence is specified in storage but not exposed through the API.**  
   `planning/PLAN.md:234` defines `chat_messages`, and `planning/PLAN.md:292` says recent history is loaded from it, but `planning/PLAN.md:270` only exposes `POST /api/chat`. Without a `GET /api/chat` or explicit "history is backend-only context" rule, a browser refresh loses visible conversation history even though the database keeps it. Add `GET /api/chat?limit=...` if persistence should be user-visible, or simplify the requirement if it is only for prompt context.

4. **Medium - Watchlist ticker validation is ambiguous across market data providers.**  
   `planning/PLAN.md:267` allows adding any `{ticker}`, while `planning/PLAN.md:150` only describes seeded simulator prices and `planning/PLAN.md:159` describes real provider polling. The current simulator code can synthesize unknown tickers, but the plan does not say whether that behavior is required, how Massive failures should behave, or what error the frontend should show for invalid symbols. Define provider-neutral behavior for unknown tickers and duplicate adds.

5. **Medium - SSE payload shape is not explicit enough.**  
   `planning/PLAN.md:178` says the server pushes "price updates for all tickers" and `planning/PLAN.md:179` lists fields, but it does not state whether each SSE message is a single update, an array, or an object keyed by ticker. The existing market stream implementation emits an object keyed by ticker. The plan should include the exact JSON shape so the frontend does not build against a different interpretation.

6. **Medium - Trade and position math leaves important edge cases open.**  
   `planning/PLAN.md:210` defines `positions`, and `planning/PLAN.md:260` defines market-order execution, but the plan does not specify weighted-average-cost calculation, fractional precision rules, whether a zero-quantity position row is deleted, or whether short selling is rejected. These details affect portfolio totals, P&L, trades, tests, and chat-executed actions.

7. **Medium - The LLM implementation reference is not self-contained.**  
   `planning/PLAN.md:284` instructs agents to use a `cerebras-inference` skill and a specific model id. That makes the plan dependent on a skill that may not exist in every agent environment, and the model identifier should be verified before implementation. Add the LiteLLM/OpenRouter configuration directly to the plan, including the exact model string, provider routing, mock behavior, timeout, and malformed-response fallback.

8. **Low - Chart baselines are unclear.**  
   `planning/PLAN.md:355` asks for daily change percent and `planning/PLAN.md:356` asks for price-over-time charts, but the simulator and SSE stream are page-load/live oriented. Define whether the baseline is seed price, first received price, previous close from Massive, or "since page load." This avoids inconsistent watchlist percentages and empty chart behavior.

9. **Low - Testing strategy needs a few API-level acceptance checks.**  
   `planning/PLAN.md:374` covers Docker and `planning/PLAN.md:424` covers E2E scenarios, but it does not pin contract tests for response schemas, SSE event shape, and validation errors. Add those tests because they protect the most likely frontend/backend integration failures.

## Strengths

- The plan has a clear single-container architecture and sensible constraints for a course capstone.
- The market-data abstraction is well scoped, and the simulator-first default is practical for demos and tests.
- The UX requirements are specific enough for a frontend agent to build the main workstation without needing a marketing-page detour.
- The testing section covers the core user workflows and correctly uses `LLM_MOCK=true` for deterministic E2E tests.

## Suggested Next Step

Before assigning more implementation work, add a compact `planning/API_CONTRACT.md` or expand section 8 with exact JSON schemas, error bodies, and SSE payload examples. That will remove the largest source of cross-agent drift.
