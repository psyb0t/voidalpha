# 01 — Foundation

Set up the monorepo, infrastructure, and base services.

## 1.1 — servicepack Init

```bash
# Clone servicepack into current directory, make it our own
# Working directory is already /home/bw/work/psyb0t/voidalpha/
git clone https://github.com/psyb0t/servicepack.git .
make own MODNAME=github.com/psyb0t/voidalpha
```

This gives us:
- Cobra CLI entry point (`cmd/main.go`)
- Service manager with auto-discovery
- Makefile with build/lint/test
- Docker build (multi-stage, static binary)

## 1.2 — PostgreSQL Schema

Core tables shared by all services:

```sql
-- Every data point from every service lands here (or in service-specific tables that reference this)
CREATE TABLE data_events (
    id              BIGSERIAL PRIMARY KEY,
    source_service  TEXT NOT NULL,          -- 'news-aggregator', 'macro-data', etc.
    category        TEXT NOT NULL,          -- 'news', 'macro', 'insider', 'options', 'sentiment', 'earnings', 'political'
    urgency         TEXT NOT NULL DEFAULT 'routine', -- 'routine', 'notable', 'alert'
    title           TEXT NOT NULL,
    body            JSONB NOT NULL,         -- service-specific payload
    source_url      TEXT,
    source_id       TEXT,                   -- dedup key from source (article ID, filing ID, etc.)
    published_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(source_service, source_id)
);

-- Tags link data_events to assets/sectors/regions
CREATE TABLE data_event_tags (
    id              BIGSERIAL PRIMARY KEY,
    event_id        BIGINT NOT NULL REFERENCES data_events(id) ON DELETE CASCADE,
    tag_type        TEXT NOT NULL,          -- 'ticker', 'sector', 'asset_class', 'region'
    tag_value       TEXT NOT NULL,          -- 'AAPL', 'technology', 'equity', 'US'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_event_tags_event ON data_event_tags(event_id);
CREATE INDEX idx_event_tags_type_value ON data_event_tags(tag_type, tag_value);

-- Market snapshots (high-frequency, separate table)
CREATE TABLE market_snapshots (
    id              BIGSERIAL PRIMARY KEY,
    ticker          TEXT NOT NULL,
    price           NUMERIC NOT NULL,
    change_pct      NUMERIC,
    volume          BIGINT,
    extra           JSONB,                  -- bid/ask, open/high/low, etc.
    captured_at     TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_snapshots_ticker_time ON market_snapshots(ticker, captured_at DESC);

-- Options flow (separate for performance)
CREATE TABLE options_flow (
    id              BIGSERIAL PRIMARY KEY,
    ticker          TEXT NOT NULL,
    expiry          DATE NOT NULL,
    strike          NUMERIC NOT NULL,
    option_type     TEXT NOT NULL,          -- 'call', 'put'
    volume          BIGINT,
    open_interest   BIGINT,
    implied_vol     NUMERIC,
    delta           NUMERIC,
    gamma           NUMERIC,
    extra           JSONB,
    captured_at     TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_options_ticker_expiry ON options_flow(ticker, expiry, captured_at DESC);

-- Computed gamma exposure
CREATE TABLE gamma_exposure (
    id              BIGSERIAL PRIMARY KEY,
    ticker          TEXT NOT NULL,
    spot_price      NUMERIC NOT NULL,
    total_gex       NUMERIC NOT NULL,      -- net gamma exposure in $ notional
    call_gex        NUMERIC NOT NULL,
    put_gex         NUMERIC NOT NULL,
    max_pain        NUMERIC,
    gex_by_strike   JSONB,                 -- [{strike, gex}, ...]
    computed_at     TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- NOTIFY trigger: fires on every insert to data_events
CREATE OR REPLACE FUNCTION notify_data_event() RETURNS trigger AS $$
BEGIN
    PERFORM pg_notify('data_events', json_build_object(
        'id', NEW.id,
        'source_service', NEW.source_service,
        'category', NEW.category,
        'urgency', NEW.urgency,
        'title', NEW.title
    )::text);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_data_event_notify
    AFTER INSERT ON data_events
    FOR EACH ROW EXECUTE FUNCTION notify_data_event();

-- Same for market_snapshots
CREATE OR REPLACE FUNCTION notify_market_snapshot() RETURNS trigger AS $$
BEGIN
    PERFORM pg_notify('market_snapshots', json_build_object(
        'id', NEW.id,
        'ticker', NEW.ticker,
        'price', NEW.price,
        'change_pct', NEW.change_pct
    )::text);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_market_snapshot_notify
    AFTER INSERT ON market_snapshots
    FOR EACH ROW EXECUTE FUNCTION notify_market_snapshot();
```

## 1.3 — Temporal Setup

Docker-compose services:
- `temporal` — Temporal server
- `temporal-admin-tools` — for namespace setup
- `temporal-ui` — web UI for monitoring workflows

Each servicepack service starts a Temporal worker internally, registering its own workflows and activities on its own task queue.
Workflows define the schedule (e.g., "fetch FRED data every hour", "check news every 5 min").

Key Temporal patterns:
- **Cron workflows** — for scheduled data fetching
- **Activities** — the actual API calls / scraping
- **Retry policies** — per-activity (e.g., 3 retries, exponential backoff for rate limits)
- **Heartbeats** — for long-running scrape activities

## 1.4 — Docker Compose

```yaml
# docker-compose.yml structure (not final, but the shape)
services:
  postgres:
    image: postgres:17
    environment:
      POSTGRES_USER: voidalpha
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: voidalpha             # default DB for the app
      POSTGRES_MULTIPLE_DATABASES: temporal  # created by init script
    volumes:
      - pg_data:/var/lib/postgresql/data
      - ./init.d:/docker-entrypoint-initdb.d  # creates voidalpha schema + temporal DB
    ports:
      - "5432:5432"

  temporal:
    image: temporalio/auto-setup:latest
    environment:
      DB: postgresql
      DB_PORT: 5432
      POSTGRES_USER: voidalpha
      POSTGRES_PWD: ${POSTGRES_PASSWORD}
      POSTGRES_SEEDS: postgres
      DBNAME: temporal                   # separate DB, not voidalpha
    depends_on:
      - postgres

  temporal-ui:
    image: temporalio/ui:latest
    environment:
      TEMPORAL_ADDRESS: temporal:7233
    ports:
      - "8080:8080"

  app:
    build: .
    env_file: .env
    environment:
      SERVICES_ENABLED: ""  # empty = all
      DATABASE_URL: postgres://voidalpha:${POSTGRES_PASSWORD}@postgres:5432/voidalpha?sslmode=disable
      TEMPORAL_ADDRESS: temporal:7233
    depends_on:
      - postgres
      - temporal
    ports:
      - "3000:3000"   # api-gateway HTTP
      - "3001:3001"   # api-gateway WebSocket
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock  # for stealthy-auto-browse + docker-claude-code

volumes:
  pg_data:
```

**Note on docker-claude-code**: Not a docker-compose service — it's spawned on-demand by the app via `docker run --rm`. The app needs Docker socket access to spin up `psyb0t/claude-code:latest` containers for summarization/categorization. Auth via `CLAUDE_CODE_OAUTH_TOKEN` env var passed to each container. Uses `--model haiku` for cost efficiency.

## 1.5 — Shared Libraries (internal/pkg/)

Before writing services, set up shared packages:

| Package | Purpose |
|---------|---------|
| `database` | PostgreSQL connection pool (pgx), raw queries for now. GORM gen later. NOTIFY listener. |
| `temporal-client` | Shared Temporal client factory, common workflow options |
| `browser` | Go client for docker-stealthy-auto-browse HTTP API |
| `llm` | Go client for docker-claude-code. Handles: export data → write temp files → mount into container → run Claude → read output files → clean up. |
| `models` | Shared data types (DataEvent, Tag, MarketSnapshot, etc.) |
| `config` | Shared config parsing via gonfiguration |
| `tagger` | Extracts tickers/sectors/regions from text (used by categorizer + individual services) |

## 1.6 — api-gateway Service

The first real service. Provides:

**Frontend API**:
- `GET /api/events?category=news&ticker=AAPL&limit=50` — query data_events
- `GET /api/snapshots?ticker=GC=F` — latest market snapshots
- `GET /api/options?ticker=SPY` — options flow
- `GET /api/gamma?ticker=SPY` — latest gamma exposure
- `WS /ws` — WebSocket, subscribes to PostgreSQL NOTIFY channels, pushes to connected clients

**Export API** (for AI processing via docker-claude-code):
- `GET /api/export/events?untagged=true&limit=50` — dump untagged events as JSON array
- `GET /api/export/events?category=news&since=2026-03-13T00:00:00Z` — dump events by filter
- `GET /api/export/event/{id}` — single event with full body
- `GET /api/export/snapshots?ticker=SPY` — latest snapshots as JSON
- `GET /api/export/options?ticker=SPY` — full options chain dump

Export endpoints return clean JSON files. The categorizer uses these to:
1. Fetch a batch of untagged events
2. Write JSON to a temp file
3. Mount that file into the docker-claude-code container
4. Prompt: "read /data/events.json, analyze each event, write results to /data/results.json"
5. Read results back, update DB

This way Claude works with files (what it's good at) instead of huge inline prompts.

Frontend connects to the WebSocket and receives events as they happen.

## 1.7 — Frontend Scaffold

Svelte app compiled to static `index.html` + JS (no SSR, no Node server in production).

Initial pages (just shells for now, real implementation in phase 07):
- `/` — Dashboard (latest events across all categories)
- `/market` — Market data + snapshots
- `/news` — News feed
- `/options` — Options flow + gamma
- `/macro` — Macro + Fed data
- `/insider` — Insider + Congress trades
- `/sentiment` — Fear & Greed + social

Served by the api-gateway service (static file serving).

## 1.8 — Deliverables

After this phase we have:
- [ ] servicepack monorepo initialized, compiles, lints, tests pass
- [ ] PostgreSQL running with schema + NOTIFY triggers
- [ ] Temporal running with UI
- [ ] Shared libraries (database, temporal-client, browser, llm, models, tagger)
- [ ] api-gateway service: REST API + WebSocket + static file serving
- [ ] Svelte frontend shell: pages routing, WebSocket connection, renders incoming events
- [ ] Docker Compose runs everything with `docker compose up`
- [ ] `.env.example` with all required variables documented
