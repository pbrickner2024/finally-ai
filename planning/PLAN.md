# FinAlly — AI Trading Workstation

## Project Specification

## 1. Vision

FinAlly (Finance Ally) is a visually stunning AI-powered trading workstation that streams live market data, lets users trade a simulated portfolio, and integrates an LLM chat assistant that can analyze positions and execute trades on the user's behalf. It looks and feels like a modern Bloomberg terminal with an AI copilot.

This is the capstone project for an agentic AI coding course. It is built entirely by Coding Agents demonstrating how orchestrated AI agents can produce a production-quality full-stack application. Agents interact through files in `planning/`.

## 2. User Experience

### First Launch

The user runs a single Docker command (or a provided start script). A browser opens to `http://localhost:8000`. No login, no signup. They immediately see:

- A watchlist of 10 default tickers with live-updating prices in a grid
- $10,000 in virtual cash
- A dark, data-rich trading terminal aesthetic
- An AI chat panel ready to assist

### What the User Can Do

- **Watch prices stream** — prices flash green (uptick) or red (downtick) with subtle CSS animations that fade
- **View sparkline mini-charts** — price action beside each ticker in the watchlist, accumulated on the frontend from the SSE stream since page load (sparklines fill in progressively)
- **Click a ticker** to see a larger detailed chart in the main chart area
- **Buy and sell shares** — market orders only, instant fill at current price, no fees, no confirmation dialog
- **Monitor their portfolio** — a heatmap (treemap) showing positions sized by weight and colored by P&L, plus a P&L chart tracking total portfolio value over time
- **View a positions table** — ticker, quantity, average cost, current price, unrealized P&L, % change
- **Chat with the AI assistant** — ask about their portfolio, get analysis, and have the AI execute trades and manage the watchlist through natural language
- **Manage the watchlist** — add/remove tickers manually or via the AI chat

### Visual Design

- **Dark theme**: backgrounds around `#0d1117` or `#1a1a2e`, muted gray borders, no pure black
- **Price flash animations**: brief green/red background highlight on price change, fading over ~500ms via CSS transitions
- **Connection status indicator**: a small colored dot (green = connected, yellow = reconnecting, red = disconnected) visible in the header
- **Professional, data-dense layout**: inspired by Bloomberg/trading terminals — every pixel earns its place
- **Responsive but desktop-first**: optimized for wide screens, functional on tablet

### Color Scheme
- Accent Yellow: `#ecad0a`
- Blue Primary: `#209dd7`
- Purple Secondary: `#753991` (submit buttons)

## 3. Architecture Overview

### Single Container, Single Port

```
┌─────────────────────────────────────────────────┐
│  Docker Container (port 8000)                   │
│                                                 │
│  FastAPI (Python/uv)                            │
│  ├── /api/*          REST endpoints             │
│  ├── /api/stream/*   SSE streaming              │
│  └── /*              Static file serving         │
│                      (Next.js export)            │
│                                                 │
│  SQLite database (volume-mounted)               │
│  Background task: market data polling/sim        │
└─────────────────────────────────────────────────┘
```

- **Frontend**: Next.js with TypeScript, built as a static export (`output: 'export'`), served by FastAPI as static files
- **Backend**: FastAPI (Python), managed as a `uv` project
- **Database**: SQLite, single file at `db/finally.db`, volume-mounted for persistence
- **Real-time data**: Server-Sent Events (SSE) — simpler than WebSockets, one-way server→client push, works everywhere
- **AI integration**: LiteLLM → OpenRouter (Cerebras for fast inference), with structured outputs for trade execution
- **Market data**: Environment-variable driven — simulator by default, real data via Massive API if key provided

### Why These Choices

| Decision | Rationale |
|---|---|
| SSE over WebSockets | One-way push is all we need; simpler, no bidirectional complexity, universal browser support |
| Static Next.js export | Single origin, no CORS issues, one port, one container, simple deployment |
| SQLite over Postgres | No auth = no multi-user = no need for a database server; self-contained, zero config |
| Single Docker container | Students run one command; no docker-compose for production, no service orchestration |
| uv for Python | Fast, modern Python project management; reproducible lockfile; what students should learn |
| Market orders only | Eliminates order book, limit order logic, partial fills — dramatically simpler portfolio math |

---

## 4. Directory Structure

```
finally/
├── frontend/                 # Next.js TypeScript project (static export)
├── backend/                  # FastAPI uv project (Python)
│   └── db/                   # Schema definitions, seed data, migration logic
├── planning/                 # Project-wide documentation for agents
│   ├── PLAN.md               # This document
│   └── ...                   # Additional agent reference docs
├── scripts/
│   ├── start_mac.sh          # Launch Docker container (macOS/Linux)
│   ├── stop_mac.sh           # Stop Docker container (macOS/Linux)
│   ├── start_windows.ps1     # Launch Docker container (Windows PowerShell)
│   └── stop_windows.ps1      # Stop Docker container (Windows PowerShell)
├── test/                     # Playwright E2E tests + docker-compose.test.yml
├── db/                       # Volume mount target (SQLite file lives here at runtime)
│   └── .gitkeep              # Directory exists in repo; finally.db is gitignored
├── Dockerfile                # Multi-stage build (Node → Python)
├── docker-compose.yml        # Optional convenience wrapper
├── .env                      # Environment variables (gitignored, .env.example committed)
└── .gitignore
```

### Key Boundaries

- **`frontend/`** is a self-contained Next.js project. It knows nothing about Python. It talks to the backend via `/api/*` endpoints and `/api/stream/*` SSE endpoints. Internal structure is up to the Frontend Engineer agent.
- **`backend/`** is a self-contained uv project with its own `pyproject.toml`. It owns all server logic including database initialization, schema, seed data, API routes, SSE streaming, market data, and LLM integration. Internal structure is up to the Backend/Market Data agents.
- **`backend/db/`** contains schema SQL definitions and seed logic. The backend lazily initializes the database on first request — creating tables and seeding default data if the SQLite file doesn't exist or is empty.
- **`db/`** at the top level is the runtime volume mount point. The SQLite file (`db/finally.db`) is created here by the backend and persists across container restarts via Docker volume.
- **`planning/`** contains project-wide documentation, including this plan. All agents reference files here as the shared contract.
- **`test/`** contains Playwright E2E tests and supporting infrastructure (e.g., `docker-compose.test.yml`). Unit tests live within `frontend/` and `backend/` respectively, following each framework's conventions.
- **`scripts/`** contains start/stop scripts that wrap Docker commands.

---

## 5. Environment Variables

```bash
# Required: OpenRouter API key for LLM chat functionality
OPENROUTER_API_KEY=your-openrouter-api-key-here

# Optional: Massive (Polygon.io) API key for real market data
# If not set, the built-in market simulator is used (recommended for most users)
MASSIVE_API_KEY=

# Optional: Set to "true" for deterministic mock LLM responses (testing)
LLM_MOCK=false
```

### Behavior

- If `MASSIVE_API_KEY` is set and non-empty → backend uses Massive REST API for market data
- If `MASSIVE_API_KEY` is absent or empty → backend uses the built-in market simulator
- If `LLM_MOCK=true` → backend returns deterministic mock LLM responses (for E2E tests)
- The backend reads `.env` from the project root (mounted into the container or read via docker `--env-file`)

---

## 6. Market Data

### Two Implementations, One Interface

Both the simulator and the Massive client implement the same abstract interface. The backend selects which to use based on the environment variable. All downstream code (SSE streaming, price cache, frontend) is agnostic to the source.

### Simulator (Default)

- Generates prices using geometric Brownian motion (GBM) with configurable drift and volatility per ticker
- Updates at ~500ms intervals
- Correlated moves across tickers (e.g., tech stocks move together)
- Occasional random "events" — sudden 2-5% moves on a ticker for drama
- Starts from realistic seed prices (e.g., AAPL ~$190, GOOGL ~$175, etc.)
- Runs as an in-process background task — no external dependencies

### Massive API (Optional)

- REST API polling (not WebSocket) — simpler, works on all tiers
- Polls for the union of all watched tickers on a configurable interval
- Free tier (5 calls/min): poll every 15 seconds
- Paid tiers: poll every 2-15 seconds depending on tier
- Parses REST response into the same format as the simulator

### Shared Price Cache

- A single background task (simulator or Massive poller) writes to an in-memory price cache
- The cache holds the latest price, previous price, and timestamp for each ticker
- SSE streams read from this cache and push updates to connected clients
- This architecture supports future multi-user scenarios without changes to the data layer

### SSE Streaming

- Endpoint: `GET /api/stream/prices`
- Long-lived SSE connection; client uses native `EventSource` API
- Server pushes price updates for all tickers known to the system at a regular cadence (~500ms) — in the single-user model this is equivalent to the user's watchlist
- Each SSE event contains ticker, price, previous price, timestamp, and change direction
- Client handles reconnection automatically (EventSource has built-in retry)

---

## 7. Database

### SQLite with Lazy Initialization

The backend checks for the SQLite database on startup (or first request). If the file doesn't exist or tables are missing, it creates the schema and seeds default data. This means:

- No separate migration step
- No manual database setup
- Fresh Docker volumes start with a clean, seeded database automatically

### Schema

All tables include a `user_id` column defaulting to `"default"`. This is hardcoded for now (single-user) but enables future multi-user support without schema migration.

**users_profile** — User state (cash balance)
- `id` TEXT PRIMARY KEY (default: `"default"`)
- `cash_balance` REAL (default: `10000.0`)
- `created_at` TEXT (ISO timestamp)

**watchlist** — Tickers the user is watching
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `ticker` TEXT
- `added_at` TEXT (ISO timestamp)
- UNIQUE constraint on `(user_id, ticker)`

**positions** — Current holdings (one row per ticker per user)
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `ticker` TEXT
- `quantity` REAL (fractional shares supported)
- `avg_cost` REAL
- `updated_at` TEXT (ISO timestamp)
- UNIQUE constraint on `(user_id, ticker)`

**trades** — Trade history (append-only log)
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `ticker` TEXT
- `side` TEXT (`"buy"` or `"sell"`)
- `quantity` REAL (fractional shares supported)
- `price` REAL
- `executed_at` TEXT (ISO timestamp)

**portfolio_snapshots** — Portfolio value over time (for P&L chart). Recorded every 30 seconds by a background task, and immediately after each trade execution.
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `total_value` REAL
- `recorded_at` TEXT (ISO timestamp)

**chat_messages** — Conversation history with LLM
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `role` TEXT (`"user"` or `"assistant"`)
- `content` TEXT
- `actions` TEXT (JSON — trades executed, watchlist changes made; null for user messages)
- `created_at` TEXT (ISO timestamp)

### Default Seed Data

- One user profile: `id="default"`, `cash_balance=10000.0`
- Ten watchlist entries: AAPL, GOOGL, MSFT, AMZN, TSLA, NVDA, META, JPM, V, NFLX

---

## 8. API Endpoints

### Market Data
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/stream/prices` | SSE stream of live price updates |

### Portfolio
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/portfolio` | Current positions, cash balance, total value, unrealized P&L |
| POST | `/api/portfolio/trade` | Execute a trade: `{ticker, quantity, side}` |
| GET | `/api/portfolio/history` | Portfolio value snapshots over time (for P&L chart) |

### Watchlist
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/watchlist` | Current watchlist tickers with latest prices |
| POST | `/api/watchlist` | Add a ticker: `{ticker}` |
| DELETE | `/api/watchlist/{ticker}` | Remove a ticker |

### Chat
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/chat` | Send a message, receive complete JSON response (message + executed actions) |

### System
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/health` | Health check (for Docker/deployment) |

---

## 9. LLM Integration

When writing code to make calls to LLMs, use cerebras-inference skill to use LiteLLM via OpenRouter to the `openrouter/openai/gpt-oss-120b` model with Cerebras as the inference provider. Structured Outputs should be used to interpret the results.

There is an OPENROUTER_API_KEY in the .env file in the project root.

### How It Works

When the user sends a chat message, the backend:

1. Loads the user's current portfolio context (cash, positions with P&L, watchlist with live prices, total portfolio value)
2. Loads recent conversation history from the `chat_messages` table
3. Constructs a prompt with a system message, portfolio context, conversation history, and the user's new message
4. Calls the LLM via LiteLLM → OpenRouter, requesting structured output, using the cerebras-inference skill
5. Parses the complete structured JSON response
6. Auto-executes any trades or watchlist changes specified in the response
7. Stores the message and executed actions in `chat_messages`
8. Returns the complete JSON response to the frontend (no token-by-token streaming — Cerebras inference is fast enough that a loading indicator is sufficient)

### Structured Output Schema

The LLM is instructed to respond with JSON matching this schema:

```json
{
  "message": "Your conversational response to the user",
  "trades": [
    {"ticker": "AAPL", "side": "buy", "quantity": 10}
  ],
  "watchlist_changes": [
    {"ticker": "PYPL", "action": "add"}
  ]
}
```

- `message` (required): The conversational text shown to the user
- `trades` (optional): Array of trades to auto-execute. Each trade goes through the same validation as manual trades (sufficient cash for buys, sufficient shares for sells)
- `watchlist_changes` (optional): Array of watchlist modifications

### Auto-Execution

Trades specified by the LLM execute automatically — no confirmation dialog. This is a deliberate design choice:
- It's a simulated environment with fake money, so the stakes are zero
- It creates an impressive, fluid demo experience
- It demonstrates agentic AI capabilities — the core theme of the course

If a trade fails validation (e.g., insufficient cash), the error is included in the chat response so the LLM can inform the user.

### System Prompt Guidance

The LLM should be prompted as "FinAlly, an AI trading assistant" with instructions to:
- Analyze portfolio composition, risk concentration, and P&L
- Suggest trades with reasoning
- Execute trades when the user asks or agrees
- Manage the watchlist proactively
- Be concise and data-driven in responses
- Always respond with valid structured JSON

### LLM Mock Mode

When `LLM_MOCK=true`, the backend returns deterministic mock responses instead of calling OpenRouter. This enables:
- Fast, free, reproducible E2E tests
- Development without an API key
- CI/CD pipelines

---

## 10. Frontend Design

### Layout

The frontend is a single-page application with a dense, terminal-inspired layout. The specific component architecture and layout system is up to the Frontend Engineer, but the UI should include these elements:

- **Watchlist panel** — grid/table of watched tickers with: ticker symbol, current price (flashing green/red on change), daily change %, and a sparkline mini-chart (accumulated from SSE since page load)
- **Main chart area** — larger chart for the currently selected ticker, with at minimum price over time. Clicking a ticker in the watchlist selects it here.
- **Portfolio heatmap** — treemap visualization where each rectangle is a position, sized by portfolio weight, colored by P&L (green = profit, red = loss)
- **P&L chart** — line chart showing total portfolio value over time, using data from `portfolio_snapshots`
- **Positions table** — tabular view of all positions: ticker, quantity, avg cost, current price, unrealized P&L, % change
- **Trade bar** — simple input area: ticker field, quantity field, buy button, sell button. Market orders, instant fill.
- **AI chat panel** — docked/collapsible sidebar. Message input, scrolling conversation history, loading indicator while waiting for LLM response. Trade executions and watchlist changes shown inline as confirmations.
- **Header** — portfolio total value (updating live), connection status indicator, cash balance

### Technical Notes

- Use `EventSource` for SSE connection to `/api/stream/prices`
- Canvas-based charting library preferred (Lightweight Charts or Recharts) for performance
- Price flash effect: on receiving a new price, briefly apply a CSS class with background color transition, then remove it
- All API calls go to the same origin (`/api/*`) — no CORS configuration needed
- Tailwind CSS for styling with a custom dark theme

---

## 11. Docker & Deployment

### Multi-Stage Dockerfile

```
Stage 1: Node 20 slim
  - Copy frontend/
  - npm install && npm run build (produces static export)

Stage 2: Python 3.12 slim
  - Install uv
  - Copy backend/
  - uv sync (install Python dependencies from lockfile)
  - Copy frontend build output into a static/ directory
  - Expose port 8000
  - CMD: uvicorn serving FastAPI app
```

FastAPI serves the static frontend files and all API routes on port 8000.

### Docker Volume

The SQLite database persists via a named Docker volume:

```bash
docker run -v finally-data:/app/db -p 8000:8000 --env-file .env finally
```

The `db/` directory in the project root maps to `/app/db` in the container. The backend writes `finally.db` to this path.

### Start/Stop Scripts

**`scripts/start_mac.sh`** (macOS/Linux):
- Builds the Docker image if not already built (or if `--build` flag passed)
- Runs the container with the volume mount, port mapping, and `.env` file
- Prints the URL to access the app
- Optionally opens the browser

**`scripts/stop_mac.sh`** (macOS/Linux):
- Stops and removes the running container
- Does NOT remove the volume (data persists)

**`scripts/start_windows.ps1`** / **`scripts/stop_windows.ps1`**: PowerShell equivalents for Windows.

All scripts should be idempotent — safe to run multiple times.

### Optional Cloud Deployment

The container is designed to deploy to AWS App Runner, Render, or any container platform. A Terraform configuration for App Runner may be provided in a `deploy/` directory as a stretch goal, but is not part of the core build.

---

## 12. Testing Strategy

### Unit Tests (within `frontend/` and `backend/`)

**Backend (pytest)**:
- Market data: simulator generates valid prices, GBM math is correct, Massive API response parsing works, both implementations conform to the abstract interface
- Portfolio: trade execution logic, P&L calculations, edge cases (selling more than owned, buying with insufficient cash, selling at a loss)
- LLM: structured output parsing handles all valid schemas, graceful handling of malformed responses, trade validation within chat flow
- API routes: correct status codes, response shapes, error handling

**Frontend (React Testing Library or similar)**:
- Component rendering with mock data
- Price flash animation triggers correctly on price changes
- Watchlist CRUD operations
- Portfolio display calculations
- Chat message rendering and loading state

### E2E Tests (in `test/`)

**Infrastructure**: A separate `docker-compose.test.yml` in `test/` that spins up the app container plus a Playwright container. This keeps browser dependencies out of the production image.

**Environment**: Tests run with `LLM_MOCK=true` by default for speed and determinism.

**Key Scenarios**:
- Fresh start: default watchlist appears, $10k balance shown, prices are streaming
- Add and remove a ticker from the watchlist
- Buy shares: cash decreases, position appears, portfolio updates
- Sell shares: cash increases, position updates or disappears
- Portfolio visualization: heatmap renders with correct colors, P&L chart has data points
- AI chat (mocked): send a message, receive a response, trade execution appears inline
- SSE resilience: disconnect and verify reconnection

---

## 13. Review Notes

### Questions & Clarifications

**Section 4 — Directory Structure**
- `.env.example` is mentioned in the `.env` line comment but is absent from the directory tree. Add it explicitly so agents know to create it.
ANSWER: Remove references to .env.example and replace them with .env

**Section 5 — Environment Variables**
- The plan says "backend reads `.env` from the project root." Inside a Docker container, the file is passed via `--env-file` and becomes plain environment variables — the process never sees the file. Clarify whether the backend should use `python-dotenv` (reads the file directly, useful for local dev outside Docker) or rely solely on `os.environ` (correct inside Docker). Both can coexist, but agents need to know which is expected.
ANSWER: use python-dotenv

**Section 6 — Market Data**
- "Server pushes price updates for all tickers known to the system" — what does "known to the system" mean exactly? Is it strictly the watchlist table? If a user adds a new ticker to the watchlist mid-session, when does the simulator (or Massive poller) begin generating prices for it?
- With Massive free tier (15s polling), the price flash animations and sparklines will update far less frequently than the simulator's 500ms. Should the plan note this UX degradation, or should the frontend handle both cadences gracefully?
- The simulator section mentions "daily change %" is shown in the watchlist panel (Section 10), but the simulator has no concept of a daily open price. Clarify what "daily change %" means for simulated data — change since simulator start? Since the current calendar day? Since a fixed seed price?

**Section 7 — Database**
- `portfolio_snapshots` records `total_value` every 30 seconds indefinitely. Over days of use this table will grow without bound. Should there be a simple retention policy (e.g., keep the last 24 hours of snapshots)?
- Is `total_value` defined as `cash_balance + sum(quantity * current_price)` for all positions? Making this explicit prevents divergent implementations.
- The `users_profile` table name is grammatically awkward for a singular row. `user_profile` is cleaner (minor, but agents will copy this name everywhere).

**Section 8 — API Endpoints**
- `GET /api/portfolio` is described as returning "positions, cash balance, total value, unrealized P&L" but the exact response shape is not defined. Without a sample JSON shape, agents will invent different field names. Recommend adding a minimal example response for at least this endpoint (and `/api/watchlist`).
- `GET /api/watchlist` says "current watchlist tickers with latest prices" — but this implies the backend merges price-cache data into the DB response. This cross-cutting concern should be called out explicitly, since agents implementing the route need to know to pull from the in-memory cache.

**Section 9 — LLM Integration**
- No limit is specified on how many messages are loaded from `chat_messages` for context. Over a long session this becomes an unbounded prompt. Recommend specifying a cap (e.g., last 20 messages).
- The LLM mock response format is not defined. Agents implementing `LLM_MOCK=true` will invent different shapes. Add a single canonical mock response object.
- Step 4 says "using the cerebras-inference skill" — but this is an agent authoring instruction, not a code pattern. Clarify what the actual Python call looks like (LiteLLM model string, base URL, etc.) so the Backend agent has a concrete target.

**Section 10 — Frontend Design**
- No mention of local development setup: when running Next.js dev server (`npm run dev` on port 3000) and FastAPI separately (port 8000), `/api/*` calls from the browser will hit port 3000 and fail. Agents need to know whether to configure a Next.js `rewrites` proxy in `next.config.js`, or whether local dev is always done inside Docker.

**Section 11 — Docker**
- The Dockerfile copies "frontend build output into a static/ directory" without specifying the exact source path (`frontend/out/` for `output: 'export'`) or target path inside the image. Agents need both paths to write the `COPY` instruction correctly.
- No HEALTHCHECK instruction is mentioned in the Dockerfile, yet Section 8 includes a `/api/health` endpoint. These should be connected.

---

### Simplification Opportunities

1. **UUID primary keys on `watchlist` and `positions`** — these tables already have a `UNIQUE(user_id, ticker)` constraint that serves as a natural key. The UUID `id` PK adds boilerplate to every query without benefit in a single-user app. Consider dropping `id` from these two tables and promoting the composite key to PRIMARY KEY.

2. **Frontend unit tests** — React Testing Library setup for a static Next.js export adds significant scaffolding (jest, jsdom, mocks) for tests that largely duplicate the E2E coverage. For a capstone demo, omitting frontend unit tests and relying on E2E tests for critical paths would reduce build complexity meaningfully.

3. **Start/stop scripts vs. docker-compose** — four script files (two platforms × two operations) maintaining the same Docker flags is fragile. A single `docker-compose.yml` with named volume, port mapping, and `env_file` covers both platforms and is a single source of truth. Consider making `docker-compose up --build` the primary dev interface and simplifying the scripts to thin wrappers around it.

4. **`watchlist_changes` action field** — the structured output schema supports only `"add"` as an `action` but implicitly also needs `"remove"`. Add `"remove"` to the schema definition explicitly, or confirm `"add"` is the only valid value.

5. **Error response format** — FastAPI defaults to `{"detail": "..."}` for HTTP errors. The plan doesn't specify an error format, so frontend agents will handle errors inconsistently. One line stating "use FastAPI's default error format" would close this gap.

---

## 14. Additional Review Notes

### Unanswered Questions from Section 13

The following items from Section 13 still require answers before agents can implement without ambiguity:

**Section 6 — Market Data (unresolved)**
- "Known to the system" for SSE streaming: clarify that the SSE endpoint streams prices for all tickers currently in the `watchlist` table. When a ticker is added to the watchlist mid-session, the background task should begin including it in the next update cycle. The frontend does not need a separate mechanism — the SSE stream naturally includes it.
- Daily change % for simulated data: recommend defining this as change from the simulator's seed price (the starting price when the process started), displayed as "since open" or "since start" to avoid implying real daily data.
- Massive API UX degradation at 15s polling: this is worth a brief note — sparklines and flash animations will feel sluggish. The frontend should handle both cadences without code changes (the flash CSS fires on any price update regardless of frequency).

**Section 7 — Database (unresolved)**
- Snapshot retention: recommend keeping last 24 hours only, deleting older rows as part of the snapshot background task (cheap, simple, no separate job needed).
- `total_value` definition: confirm it is `cash_balance + sum(position.quantity * current_price)` for all open positions. Any divergence here causes the P&L chart and portfolio header to show different numbers.

**Section 9 — LLM Integration (unresolved)**
- Chat history cap: recommend capping at 20 messages (10 turns) to bound prompt size.
- LLM mock canonical response: a concrete example is needed. Recommend this as the canonical mock:
  ```json
  {
    "message": "I have reviewed your portfolio. You hold no positions currently. Consider buying some stocks.",
    "trades": [],
    "watchlist_changes": []
  }
  ```
- Cerebras LiteLLM call: the concrete Python pattern is `litellm.completion(model="openrouter/openai/gpt-oss-120b", messages=[...], api_base="https://openrouter.ai/api/v1", api_key=os.environ["OPENROUTER_API_KEY"])` with `response_format` set to enforce structured output.

### New Questions Not Addressed in Section 13

**Section 2 — User Experience**
- The positions table shows "% change" — is this the unrealized P&L percentage (relative to avg cost), or the daily price change percentage? These are different values. Clarify which column each label refers to.
- When a position is fully sold (quantity reaches 0), should the row be deleted from `positions` or remain with `quantity=0`? The trade execution logic and the portfolio display both depend on this being consistent.

**Section 6 — Market Data**
- The SSE event payload is described as containing "ticker, price, previous price, timestamp, and change direction." The `change direction` field (up/down/flat) is redundant — it can be derived from `price > previous_price`. Removing it from the event simplifies both backend and frontend. If kept, clarify whether "flat" is a valid direction (i.e., price unchanged) or whether only "up"/"down" are emitted.
- The price cache holds "latest price, previous price, and timestamp." Is this per-ticker? Confirm the data structure is a dict keyed by ticker symbol (e.g., `{"AAPL": {"price": 192.5, "prev_price": 191.8, "timestamp": "..."}}`). Without this, backend and SSE agents may structure it differently.

**Section 7 — Database**
- `trades` table has no index defined. With frequent trading activity the `WHERE user_id = 'default'` scans will be fine for SQLite at low volume, but if agents add `ORDER BY executed_at DESC` queries for history they should add an index. Mention whether indexes are in scope or left to the backend agent.
- `chat_messages` stores `actions` as a JSON string in a TEXT column. No example JSON is given. Provide a minimal example (e.g., `{"trades": [{"ticker": "AAPL", "side": "buy", "quantity": 10, "price": 192.5}], "watchlist_changes": []}`). Without this, the backend agent and frontend will serialize/deserialize different shapes.

**Section 8 — API Endpoints**
- `POST /api/portfolio/trade` accepts `{ticker, quantity, side}`. What HTTP status code is returned on a validation failure (e.g., insufficient funds, invalid ticker, selling more than owned)? If it is 400 with `{"detail": "..."}`, say so. If it is 200 with an error field in the body, that is a different pattern.
- `GET /api/portfolio/history` — no time range parameters are defined. Does it return all snapshots, or a fixed window (e.g., last 24 hours)? If the snapshot retention policy from Section 7 is adopted, this naturally limits the response, but it should be stated explicitly.
- `GET /api/watchlist` response: does it include tickers for which no price is yet available in the cache (e.g., a ticker just added but not yet polled)? If yes, what value is returned for price — `null`, `0`, or the ticker is omitted until a price is available?
- There is no `GET /api/chat` endpoint for loading conversation history. On page refresh, the chat panel would appear empty. Either add a history endpoint or explicitly state that chat history is not persisted across page loads in the UI (even though it is stored in the DB).

**Section 9 — LLM Integration**
- The system prompt instructs the LLM to "execute trades when the user asks or agrees." There is no mechanism for the LLM to ask for confirmation before executing. This is consistent with the auto-execution design, but the system prompt guidance should be explicit: the LLM must never ask "shall I execute this?" — it should execute and report.
- What happens if the LLM returns a `trades` array with a ticker that is not in the watchlist? Does the backend add it to the watchlist automatically, reject the trade, or execute and leave the watchlist unchanged? This needs to be explicit.
- Structured output schema: `quantity` type is not specified. Is it an integer or a float? Section 7 says "fractional shares supported" in the `positions` table, but the LLM schema should match. Confirm `quantity` in structured output is a number (float), not constrained to integer.

**Section 10 — Frontend Design**
- The AI chat panel is described as "docked/collapsible." Clarify what "collapsed" state shows — just a tab/button, or does it hide completely? This affects the overall layout grid.
- Sparklines "accumulate from SSE stream since page load." How many data points should be shown? 50? 100? Is there a maximum? Without a cap, long-running sessions accumulate unbounded arrays in browser memory per ticker.
- The trade bar has a "ticker field." Is this a free-text input or a dropdown/autocomplete from the current watchlist? Free text allows buying tickers not on the watchlist, which may be desirable or undesirable — clarify.

**Section 11 — Docker**
- The `--env-file .env` flag in the `docker run` example assumes the command is run from the project root. The start scripts should either enforce this (e.g., `cd` to script directory's parent) or use an absolute path. Without this, users running the script from a different working directory will get "env file not found" errors.
- `python-dotenv` is confirmed in Section 13 for local dev. The Dockerfile should NOT copy `.env` into the image (it is gitignored for good reason). Confirm `.env` is in `.dockerignore` and that inside Docker, the backend relies on the environment variables injected via `--env-file`, with `python-dotenv` only loading the file when it is present on disk (i.e., `load_dotenv()` with no `override=True` is safe).

**Section 12 — Testing**
- The E2E test for "SSE resilience: disconnect and verify reconnection" is non-trivial to implement in Playwright. Playwright does not expose direct network interruption for SSE; this typically requires injecting a network fault or stopping/restarting the server. Either describe how this test is expected to work mechanically, or mark it as a stretch goal.
- No mention of test data isolation: E2E tests that execute trades will mutate the database. If tests run sequentially against a shared container, later tests may see state from earlier ones. Clarify whether tests reset the DB between runs (e.g., by restarting the container or calling a reset endpoint) or are written to be order-independent.

### Additional Simplification Opportunities

6. **`change_direction` in SSE event** — this field is derivable from `price` and `previous_price`. Removing it reduces the event payload and eliminates a class of bugs (direction disagreeing with the prices). The frontend can compute direction inline.

7. **`actions` JSON column in `chat_messages`** — since the backend already has structured output from the LLM, storing `actions` as a raw JSON string in SQLite is a serialization round-trip with no query benefit. For a single-user app where chat history is never queried by action type, this is fine, but confirm agents know to `json.dumps` on write and `json.loads` on read (or use a Pydantic model). A note here would prevent subtle bugs.

8. **`GET /api/chat` history endpoint** — if the frontend loads chat history on startup (to repopulate the panel after a refresh), a simple `GET /api/chat?limit=50` endpoint using the same `chat_messages` table avoids duplicating the data fetch pattern. If this endpoint is not planned, explicitly state that the chat panel starts empty on every page load, so the Frontend agent does not build an unnecessary fetch.

9. **Background task for portfolio snapshots** — two background tasks are implied: one for market data (simulator or Massive poller) and one for portfolio snapshots. Both run on intervals. These should use the same mechanism (FastAPI `lifespan` with `asyncio.create_task`) rather than mixing `BackgroundTasks` (per-request) and startup events. Make the task mechanism explicit so the backend agent does not mix patterns.

10. **Ticker validation** — `POST /api/watchlist` and `POST /api/portfolio/trade` both accept a `ticker` string. There is no mention of whether the backend validates that the ticker is a real (or at least plausible) symbol. For the simulator, any string is valid (it will just start generating prices for it). For the Massive API, an invalid ticker will return no data. A simple length/format check (e.g., 1-5 uppercase letters) would prevent garbage entries without adding significant complexity.
