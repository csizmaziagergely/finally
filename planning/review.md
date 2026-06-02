# PLAN.md Review — FinAlly AI Trading Workstation

Reviewed against: `planning/PLAN.md` (commit `6b568a9`)
Reviewer: Claude Code (Sonnet 4.6)
Date: 2026-06-02

---

## Summary

The plan is well-structured and largely complete for a capstone project. The architecture choices are sound and well-justified. There are several gaps and ambiguities that would cause agents to make inconsistent or blocking decisions without further guidance. The main problem areas are: SSE fan-out under concurrent browser tabs, the lazy DB init race condition, the LLM mock response contract, and a handful of underspecified frontend interactions.

---

## 1. Completeness Gaps

### 1.1 SSE fan-out is unspecified

Section 6 says "a single background task writes to an in-memory price cache" and "SSE streams read from this cache." It does not say how multiple simultaneous SSE connections (e.g., two browser tabs) share the cache. An agent building the SSE endpoint needs to decide whether to use asyncio queues, an async generator polling the cache on a timer, or a pub/sub broadcast. Without guidance, two agents building the background task and the SSE endpoint will make incompatible assumptions.

**Recommendation**: Specify the push model explicitly. The simplest approach is: the SSE endpoint polls the in-memory dict every 500ms in an async generator and yields the full watchlist snapshot each tick. State this in section 6.

### 1.2 Lazy DB initialization has a race condition

Section 7 says the backend "checks for the SQLite database on startup (or first request)." If "on first request" is the chosen approach and two concurrent requests arrive before init completes, both will attempt to create tables and seed data simultaneously. SQLite is not safe for concurrent DDL from multiple threads/coroutines without explicit locking.

**Recommendation**: Clarify that init runs once at application startup (in a FastAPI `lifespan` handler), not on the first request. This eliminates the race entirely.

### 1.3 `GET /api/watchlist` response shape is unspecified

The endpoint table says it returns "current watchlist tickers with latest prices" but does not define the JSON shape. Agents building the frontend and backend will produce incompatible structures unless this is pinned. The same gap applies to `GET /api/portfolio`.

**Recommendation**: Add a short response schema example for each endpoint, even if informal. For watchlist: `[{"ticker": "AAPL", "price": 191.50, "prev_price": 190.00, "change_pct": 0.79}]`. For portfolio: define whether positions are a list or a keyed object, and what fields are included.

### 1.4 The LLM mock response contract is undefined

Section 9 says `LLM_MOCK=true` returns "deterministic mock responses" but never specifies what those responses are. The E2E test section says "AI chat (mocked): send a message, receive a response, trade execution appears inline." For this test to be deterministic, the mock must return a specific known response to a known input. If the backend agent and the test agent each invent their own mock behavior, the test will fail.

**Recommendation**: Define at minimum one canonical mock scenario: e.g., any user message receives `{"message": "I have executed a mock trade.", "trades": [{"ticker": "AAPL", "side": "buy", "quantity": 1}], "watchlist_changes": []}`. Commit this as a fixture or document it here.

### 1.5 No error response schema defined

The API section documents happy-path responses only. It does not specify how errors are returned (HTTP status codes, JSON shape of error bodies). Agents will independently choose between `{"error": "..."}`, `{"detail": "..."}` (FastAPI default), and other conventions, causing frontend error handling to be inconsistent.

**Recommendation**: State that errors follow FastAPI's default `{"detail": "..."}` shape and specify which HTTP codes are used per endpoint (e.g., 400 for invalid ticker, 422 for validation errors, 409 for duplicate watchlist entry, 400 for insufficient funds).

### 1.6 Watchlist validation is absent

`POST /api/watchlist` accepts `{ticker}` but the plan says nothing about validating whether the ticker is real. With the simulator, any string is valid. With the Massive API, an invalid ticker would either fail silently or cause polling errors. There is no guidance on whether to validate tickers at add time or let them silently fail.

**Recommendation**: State the policy explicitly. Simplest for capstone: accept any non-empty uppercase alphabetic string of 1-5 characters; return 400 otherwise. With the simulator, all tickers immediately start generating prices after being added.

### 1.7 Removing a position when quantity hits zero

When a sell trade reduces a position's quantity to exactly zero, should the row in the `positions` table be deleted or left with `quantity=0`? The plan does not specify. The frontend positions table and portfolio heatmap will behave differently depending on which choice is made.

**Recommendation**: Specify "delete the row when quantity reaches zero."

### 1.8 The `portfolio_snapshots` background task interval conflicts with the SSE stream

Section 7 says snapshots are recorded "every 30 seconds by a background task." The SSE stream pushes prices every ~500ms. If the P&L chart on the frontend uses the `portfolio_snapshots` API (which has 30-second resolution), it will feel static compared to the live-updating header value. There is no guidance on whether the header's "total portfolio value" is computed client-side from live prices or fetched from the snapshots endpoint.

**Recommendation**: Clarify that the header value is computed client-side by the frontend using live SSE prices plus the positions data (fetched once on load). The `portfolio_snapshots` endpoint is only for the historical P&L line chart.

---

## 2. Technical Risks and Design Concerns

### 2.1 Static Next.js export breaks dynamic routing

`output: 'export'` in Next.js produces a fully static HTML/JS bundle. This works fine for a single-page app with no server-side rendering. However, if any agent adds Next.js API routes (`/pages/api/` or `app/api/`) or uses `getServerSideProps`, the build will fail silently or at export time. This is a footgun for agents unfamiliar with the constraint.

**Recommendation**: Add an explicit note in the frontend section: "Do not use Next.js API routes or server-side data fetching. All data comes from the FastAPI backend via client-side fetch or EventSource."

### 2.2 FastAPI static file serving with Next.js export path collisions

FastAPI serves the Next.js static export at `/*`. FastAPI also serves `/api/*`. If the Next.js build produces a file at `out/api/index.html` (e.g., a page route named `api`), it would shadow the FastAPI API routes. This is unlikely but possible if an agent creates a Next.js page called `api`.

**Recommendation**: Explicitly note that the `/api` path prefix is reserved for the backend. Mount static files after API routes so the API takes precedence.

### 2.3 SQLite concurrent write contention with multiple background tasks

Three things write to SQLite concurrently: the portfolio snapshots background task (every 30s), trade execution (on demand), and chat message storage (on demand). SQLite in WAL mode handles concurrent reads well but serializes writes. For a single-user capstone this is fine in practice, but agents should enable WAL mode (`PRAGMA journal_mode=WAL`) at init time to avoid "database is locked" errors during demo if a snapshot write races a trade.

**Recommendation**: Add `PRAGMA journal_mode=WAL` and `PRAGMA busy_timeout=5000` to the database initialization sequence.

### 2.4 The GBM simulator adds tickers to the price cache only if they are in the watchlist

Section 6 says the SSE stream "pushes price updates for all tickers known to the system." If a user adds a ticker to the watchlist, the simulator needs to start generating prices for it. The plan does not specify how the simulator learns about new tickers added at runtime (vs. only the seed list). This is a non-trivial integration point.

**Recommendation**: State that the simulator reads the watchlist from the database (or a shared in-memory set) each tick and generates prices for all entries, including newly added ones. A simple approach: the simulator holds a set of active tickers in memory; the watchlist add/remove API endpoints update this set directly.

### 2.5 LLM latency and the "no streaming" choice

Section 9 says "no token-by-token streaming — Cerebras inference is fast enough that a loading indicator is sufficient." This is a reasonable tradeoff, but Cerebras via OpenRouter can still take 3-8 seconds for a complex structured output request. A loading indicator with no partial feedback may feel unresponsive compared to streaming competitors.

This is a deliberate product choice, not a bug. However, agents should be aware the frontend must show a clear loading state (spinner or "thinking..." text) from the moment the user submits until the full response arrives. Without explicit guidance, a frontend agent might omit this and the chat will appear broken during latency spikes.

**Recommendation**: Explicitly call out in section 10 (Frontend) that the chat panel must show a loading indicator from POST /api/chat submission until response receipt.

---

## 3. Over-engineered for a Capstone

### 3.1 Correlated GBM moves across tickers

Section 6 mentions "correlated moves across tickers (e.g., tech stocks move together)." Implementing true correlation in GBM requires a Cholesky decomposition of a covariance matrix. This is correct quantitative finance but is significant complexity for a simulator that exists only to make prices look interesting on screen.

**Recommendation**: Simplify to independent GBM per ticker with a shared market "mood" factor (a single random multiplier applied to all tickers each tick). The visual result is similar, the code is 10 lines instead of 50, and it avoids a potential implementation bug that would be hard to test.

### 3.2 `user_id` column on every table

The plan adds `user_id TEXT DEFAULT "default"` to every table "for future multi-user support." For a capstone with one hardcoded user, this adds foreign key surface area, seed data complexity, and confusion about when to filter by user_id in queries. Any future multi-user migration would need authentication anyway, which is a much larger change than adding a column.

**Recommendation**: For the capstone, omit `user_id` entirely and document "multi-user not supported" in the plan. Add a note that this would be the first schema change for multi-user. This reduces query complexity and removes a class of bugs where agents forget to filter by user_id.

*Note: This is a mild concern — the existing choice is not wrong, just adds friction.*

---

## 4. Under-engineered for the Goals

### 4.1 No guidance on chat conversation history length

Section 9 says "loads recent conversation history from the chat_messages table" but does not define "recent." With no limit, a long-running demo will eventually exceed the LLM's context window. With too short a limit, the AI loses conversational context.

**Recommendation**: Specify a concrete limit, e.g., "last 20 messages (10 exchanges)." This is a single number that agents will otherwise each choose independently.

### 4.2 No rate limiting on trade execution

The `POST /api/portfolio/trade` endpoint has no throttle. A frontend bug or LLM returning a large trades array could flood the endpoint. For a demo environment this is acceptable, but an AI that executes 50 trades in one chat response should probably be bounded.

**Recommendation**: Add a note in section 9 (LLM) that the backend should limit auto-execution to at most 5 trades per chat message. Return a warning in the response if the LLM requested more.

### 4.3 The `daily change %` in the watchlist panel has no data source

Section 10 lists "daily change %" in the watchlist panel. This implies a known opening price or previous close. The simulator has no concept of "day" — it generates continuous GBM from seed prices. The SSE stream carries `prev_price` (the previous tick, ~500ms ago), not the price at market open.

**Recommendation**: Either remove "daily change %" from the UI spec and replace it with "change since page load" (which is computable from the SSE stream), or define a "session open price" as the first price seen for each ticker after page load. Clarify which interpretation agents should implement.

---

## 5. Ambiguities That Would Block Agent Implementation

| # | Location | Ambiguity | Blocking? |
|---|----------|-----------|-----------|
| A | Section 6, SSE | How does the SSE endpoint deliver updates — polling cache on a timer, or push from background task via queue? | Yes — two agents will build incompatible halves |
| B | Section 8, API | Response JSON shape for `/api/portfolio` and `/api/watchlist` | Yes — frontend and backend will be mismatched |
| C | Section 9, Mock | What does the mock LLM return for E2E tests? | Yes — E2E tests will not pass |
| D | Section 7, positions | Delete row or set quantity=0 when a full sell is executed? | Yes — frontend display logic differs |
| E | Section 6, simulator | How does the simulator learn about dynamically added tickers? | Yes — newly added tickers will never stream prices |
| F | Section 10, chart | What data powers the main chart area — SSE history buffered client-side, or a separate OHLCV endpoint? | Yes — a separate endpoint does not exist in the API table |
| G | Section 10, watchlist | "Daily change %" — from what baseline price? | Moderate — agents will implement inconsistently |
| H | Section 9, history | How many recent chat messages are included in LLM context? | Moderate — each agent will choose a different default |

### Note on F (main chart data)

The plan says users can "click a ticker to see a larger detailed chart." The main chart shows "price over time." The API table has no OHLCV or price history endpoint. The SSE stream only carries current prices. This means the main chart can only show data accumulated since page load via the SSE stream (same mechanism as sparklines, just rendered larger). Agents should confirm this is the intended behavior, or a new endpoint needs to be added.

---

## Conclusion

The plan is solid and complete enough to start implementation. The five "Yes — blocking" ambiguities in section 5 should be resolved before backend and frontend agents diverge. The most important changes, in priority order:

1. Define the SSE delivery model (polling cache vs. pub/sub queue) in section 6.
2. Add response JSON examples for `/api/portfolio` and `/api/watchlist` in section 8.
3. Define the LLM mock response fixture in section 9.
4. Clarify that "daily change %" means "change since page load" or add a session-open-price concept.
5. Specify that DB init happens at startup (lifespan handler), not on first request.
