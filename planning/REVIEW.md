# Review of Changes Since `HEAD`

## Findings

1. **High — The documented SSE wire format does not match the endpoint.**  
   `planning/MARKET_INTERFACE.md:256-263` says every SSE event contains one flat `PriceUpdate` object and that clients should accumulate those objects. The implemented endpoint instead sends one object keyed by ticker containing the complete cache snapshot (`backend/app/market/stream.py:75-83`), for example `{"AAPL": {...}, "TSLA": {...}}`. A client built from the new contract will look for `event.ticker` and receive `undefined` on every message. Document the actual snapshot envelope, or change the endpoint and its tests to emit the per-ticker events promised here.

2. **High — The Massive last-trade timestamp unit is contradictory, and the production adapter follows the unsafe interpretation.**  
   The Python example labels `snap.last_trade.timestamp` as Unix milliseconds at `planning/MASSIVE_API.md:73-79`, while the same document identifies that field as nanoseconds and instructs division by `1e9` at `planning/MASSIVE_API.md:122-144`. The raw example value is also nanosecond-sized. Meanwhile, `backend/app/market/massive_client.py:101-107` divides the value by only `1000`, producing timestamps roughly one million times too large if the documented API response is used. Resolve the documentation contradiction and update the adapter to normalize the actual SDK unit; otherwise Massive-backed SSE events carry unusable dates.

3. **Medium — The documented dynamic-add behavior is false for the Massive source.**  
   `planning/MARKET_INTERFACE.md:200-207` states that `add_ticker()` updates the cache in the same call and that the cache immediately has a seed price. That is only true for `SimulatorDataSource`; `MassiveDataSource.add_ticker()` merely appends to its polling list, so the cache remains empty until the next successful poll (up to 15 seconds by default, or indefinitely after an API failure/invalid symbol). Make the example explicitly simulator-only and document the Massive delay so REST handlers and UI code do not assume a price is immediately available.

4. **Medium — The removal/SSE contract can leave deleted tickers permanently visible in the documented client model.**  
   `planning/MARKET_INTERFACE.md:209-211` says removal makes the SSE stream stop emitting the ticker, while `planning/MARKET_INTERFACE.md:263` tells the client to accumulate per-ticker messages. `PriceCache.remove()` does not increment `version`, so removal itself emits no event; nor does the stream send a tombstone. With the Massive source, no later event is guaranteed if polling fails or the last ticker was removed. Even after a later full-snapshot event, an accumulating client has no specified deletion signal. Define snapshot-replacement semantics or emit an explicit removal event, and increment the cache version on removal.

5. **Low — Host-local metadata is included in the proposed change set.**  
   `.DS_Store` changed as an opaque binary, and `.claude/settings.local.json:2-16` adds author-machine web permissions and sandbox preferences. Both files are currently tracked, so incidental local state will continue appearing in commits. Unless these settings are intentionally required for every contributor, restore these two changes and add `.DS_Store` plus `.claude/settings.local.json` to `.gitignore` (removing them from tracking in a dedicated cleanup).

## Verification Notes

- Reviewed all tracked and untracked changes reported by `git status` against `HEAD`.
- Cross-checked the new interface and API documents against the current market-data implementation.
- No application code changed in this working tree, so tests were not required to evaluate the documentation-only behavior claims.
