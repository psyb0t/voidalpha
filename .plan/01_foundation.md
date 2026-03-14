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
- **Go 1.25**, Docker-based build (static binary via `golang:1.25-alpine`)
- Cobra CLI entry point (`cmd/main.go`) with `pkg/runner` for signal handling + graceful shutdown
- `cmd/init.go` — custom initialization hook (never touched by framework updates). Perfect for adding Loki slog handler.
- Service manager with auto-discovery (`gofindimpl` generates `services.gen.go`)
- Split Makefile: `Makefile` (ours, never touched) + `Makefile.servicepack` (framework, auto-updated)
- Split Dockerfiles: `Dockerfile`/`Dockerfile.dev` (ours) + `Dockerfile.servicepack`/`Dockerfile.servicepack.dev` (framework)
- Framework update system: `make servicepack-update` → review → merge/revert

**Logging**: `log/slog` with [`slog-configurator`](https://github.com/psyb0t/slog-configurator). All services use `slog.Info/Error/Warn/Debug`. Custom handlers (Loki, etc.) registered in `cmd/init.go`.

**Service interface**:
```go
type Service interface {
    Name() string
    Run(ctx context.Context) error
    Stop(ctx context.Context) error
}
```

Every service exports a `ServiceName` const and a `New() (*Type, error)` constructor. The service manager auto-discovers implementations and generates registration code.

**Optional interfaces** (implement any combo):
```go
// Retryable — auto-restart on failure
type Retryable interface {
    MaxRetries() int
    RetryDelay() time.Duration
}

// AllowedFailure — service can die without killing everything
type AllowedFailure interface {
    IsAllowedFailure() bool
}

// Dependent — service waits for dependencies to start first
type Dependent interface {
    Dependencies() []string  // other service names
}
```

The service manager resolves dependencies via topological sort, starts services in groups (same group = concurrent), shuts down in reverse order, and recovers panics.

**Core deps**: `ctxerrors` (error wrapping), `goenv` (dev/prod detection), `gonfiguration` (env var config parsing), `slog-configurator` (log setup), `cobra` (CLI).

**Makefile commands**: `make service NAME=x` (create), `make service-remove NAME=x`, `make build`, `make lint`/`make lint-fix`, `make test`/`make test-coverage` (90% min), `make dep`, `make run-dev`, `make backup`/`make backup-restore`.

## 1.2 — PostgreSQL Schema

All tables use UUID primary keys and TIMESTAMPTZ for timestamps. Each service writes structured data to its own dedicated table, then publishes a NATS JetStream notification. Deduplication happens at the dedicated table level (each has its own UNIQUE constraint).

```sql
---
-- MARKET DATA (Phase 02)
---

-- High-frequency price snapshots
CREATE TABLE market_snapshots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ticker          TEXT NOT NULL,
    price           NUMERIC NOT NULL,
    change_pct      NUMERIC,
    volume          BIGINT,
    extra           JSONB,                  -- bid/ask, open/high/low, etc.
    captured_at     TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_snapshots_ticker_time ON market_snapshots(ticker, captured_at DESC);

-- Options chain data
CREATE TABLE options_flow (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
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
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ticker          TEXT NOT NULL,
    spot_price      NUMERIC NOT NULL,
    total_gex       NUMERIC NOT NULL,
    call_gex        NUMERIC NOT NULL,
    put_gex         NUMERIC NOT NULL,
    max_pain        NUMERIC,
    gex_by_strike   JSONB,                 -- [{strike, gex}, ...]
    computed_at     TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Earnings reports and calendar
CREATE TABLE earnings_reports (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ticker          TEXT NOT NULL,
    report_date     DATE NOT NULL,
    hour            TEXT,                   -- 'bmo', 'amc', 'dmh'
    quarter         INT,
    year            INT,
    eps_estimate    NUMERIC,
    eps_actual      NUMERIC,
    revenue_estimate NUMERIC,
    revenue_actual  NUMERIC,
    surprise_pct    NUMERIC,
    source_id       TEXT UNIQUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_earnings_ticker ON earnings_reports(ticker, report_date DESC);

---
-- NEWS & SENTIMENT (Phase 03)
---

CREATE TABLE news_articles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_name     TEXT NOT NULL,          -- 'Reuters', 'CNBC', 'Finnhub', etc.
    headline        TEXT NOT NULL,
    summary         TEXT,
    url             TEXT,
    image_url       TEXT,
    body_text       TEXT,
    published_at    TIMESTAMPTZ,
    source_id       TEXT UNIQUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_news_published ON news_articles(published_at DESC);

CREATE TABLE youtube_transcripts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    channel         TEXT NOT NULL,
    channel_id      TEXT NOT NULL,
    video_id        TEXT NOT NULL UNIQUE,
    title           TEXT NOT NULL,
    transcript      TEXT NOT NULL,
    duration_secs   INT,
    published_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_youtube_channel ON youtube_transcripts(channel_id, published_at DESC);

CREATE TABLE sentiment_readings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    indicator       TEXT NOT NULL,          -- 'fear_greed', 'put_call', 'vix', 'aaii', 'reddit'
    value           NUMERIC NOT NULL,
    label           TEXT,                   -- 'extreme_fear', 'fear', 'neutral', 'greed', etc.
    components      JSONB,
    captured_at     TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_sentiment_indicator ON sentiment_readings(indicator, captured_at DESC);

---
-- MACRO & FED (Phase 04)
---

CREATE TABLE macro_indicators (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    series_id       TEXT NOT NULL,          -- 'GDP', 'CPIAUCSL', 'UNRATE', etc.
    value           NUMERIC NOT NULL,
    previous_value  NUMERIC,
    change_pct      NUMERIC,
    yoy_pct         NUMERIC,
    source          TEXT NOT NULL,          -- 'fred', 'bls', 'treasury'
    observed_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_macro_series ON macro_indicators(series_id, observed_at DESC);

CREATE TABLE fed_speeches (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    speaker         TEXT NOT NULL,
    speech_type     TEXT NOT NULL,          -- 'speech', 'testimony', 'press_release', 'minutes'
    title           TEXT NOT NULL,
    url             TEXT,
    full_text       TEXT,
    published_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_fed_speeches_date ON fed_speeches(published_at DESC);

CREATE TABLE fomc_meetings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_start   DATE NOT NULL UNIQUE,
    meeting_end     DATE NOT NULL,
    minutes_release DATE,
    press_conference BOOLEAN DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE rate_expectations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_date    DATE NOT NULL,
    hold_pct        NUMERIC,
    cut_25_pct      NUMERIC,
    cut_50_pct      NUMERIC,
    hike_25_pct     NUMERIC,
    shift_from_prev NUMERIC,
    captured_at     TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_rate_exp_meeting ON rate_expectations(meeting_date, captured_at DESC);

CREATE TABLE econ_calendar (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_name      TEXT NOT NULL,
    country         TEXT NOT NULL DEFAULT 'US',
    scheduled_at    TIMESTAMPTZ NOT NULL,
    impact          TEXT NOT NULL,          -- 'low', 'medium', 'high'
    previous        TEXT,
    forecast        TEXT,
    source_id       TEXT UNIQUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_econ_cal_scheduled ON econ_calendar(scheduled_at);

---
-- INSTITUTIONAL & POLITICAL (Phase 05)
---

CREATE TABLE congress_trades (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    member          TEXT NOT NULL,
    party           TEXT NOT NULL,          -- 'D', 'R', 'I'
    chamber         TEXT NOT NULL,          -- 'House', 'Senate'
    ticker          TEXT NOT NULL,
    trade_type      TEXT NOT NULL,          -- 'purchase', 'sale'
    amount_range    TEXT NOT NULL,
    trade_date      DATE NOT NULL,
    filed_date      DATE,
    committees      JSONB,
    source          TEXT NOT NULL,          -- 'capitoltrades', 'quiver', 'house_clerk'
    source_id       TEXT UNIQUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_congress_ticker ON congress_trades(ticker, trade_date DESC);
CREATE INDEX idx_congress_member ON congress_trades(member, trade_date DESC);

CREATE TABLE insider_trades (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    insider_name    TEXT NOT NULL,
    role            TEXT NOT NULL,          -- 'CEO', 'CFO', 'Director', '10% Owner'
    ticker          TEXT NOT NULL,
    transaction_type TEXT NOT NULL,         -- 'purchase', 'sale', 'grant', 'exercise'
    shares          BIGINT,
    price           NUMERIC,
    total_value     NUMERIC,
    is_10b5_1       BOOLEAN DEFAULT false,
    filing_date     DATE,
    transaction_date DATE,
    source_id       TEXT UNIQUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_insider_ticker ON insider_trades(ticker, transaction_date DESC);

CREATE TABLE short_interest (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ticker          TEXT NOT NULL,
    shares_short    BIGINT NOT NULL,
    float_pct       NUMERIC,
    days_to_cover   NUMERIC,
    change_pct      NUMERIC,
    settlement_date DATE NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_short_interest_ticker ON short_interest(ticker, settlement_date DESC);

CREATE TABLE dark_pool_volume (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ticker          TEXT NOT NULL,
    ats_volume      BIGINT NOT NULL,
    total_volume    BIGINT NOT NULL,
    dark_pct        NUMERIC NOT NULL,
    top_ats         JSONB,
    week_ending     DATE NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_dark_pool_ticker ON dark_pool_volume(ticker, week_ending DESC);

CREATE TABLE ftd_data (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ticker          TEXT NOT NULL,
    quantity        BIGINT NOT NULL,
    price           NUMERIC,
    notional_value  NUMERIC,
    settlement_date DATE NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_ftd_ticker ON ftd_data(ticker, settlement_date DESC);

```

## 1.3 — NATS JetStream

Event bus for inter-service notifications. Replaces PostgreSQL LISTEN/NOTIFY. Guarantees at-least-once delivery per consumer group — if a consumer is down, messages queue up and are delivered when it reconnects.

**Streams**:

| Stream | Subjects | Retention | Description |
|--------|----------|-----------|-------------|
| `EVENTS` | `events.>` | Limits (size/age) | All service notifications |
| `SNAPSHOTS` | `snapshots.>` | Limits (short, e.g. 1h) | High-frequency market snapshot ticks |

**Subject naming**: `events.{table_name}` — e.g. `events.news_articles`, `events.congress_trades`, `events.earnings_reports`.
For snapshots: `snapshots.{ticker}` — e.g. `snapshots.SPY`, `snapshots.BTC-USD`.

**Message payload** (JSON):
```json
{"id": "uuid", "table": "congress_trades"}
```
For snapshots (inline data, no DB lookup needed):
```json
{"id": "uuid", "ticker": "SPY", "price": 520.50, "change_pct": -1.2}
```

**Consumer groups** (durable pull consumers):
Each service type that wants to receive events creates a durable consumer with a unique name. NATS guarantees each consumer group gets every message at least once.

| Consumer | Stream | Filter | Purpose |
|----------|--------|--------|---------|
| `api-gateway` | `EVENTS` | `events.>` | Push to WebSocket for frontend |
| `api-gateway` | `SNAPSHOTS` | `snapshots.>` | Push real-time prices to WebSocket |
| (future) | `EVENTS` | `events.>` | AI categorization, alerting, etc. |

**Flow**:
```
Service writes to dedicated table → publishes to NATS subject
                                        ↓
                              JetStream persists message
                                        ↓
                    Each durable consumer gets it at least once
                    (api-gateway → WebSocket → frontend)
```

**Go SDK**: `github.com/nats-io/nats.go` — official, well-maintained, JetStream support built in.

**Env Var**: `VOIDALPHA_NATS_URL` (default: `nats://nats:4222`)

## 1.4 — Temporal Setup

Internal service in the docker stack, passwordless, default namespace only. No admin tools, no extra setup.

Docker-compose services:
- `temporal` — Temporal server (auto-setup, default namespace)
- `temporal-ui` — web UI for monitoring workflows (optional, for debugging)

Each servicepack service starts a Temporal worker internally, registering workflows and functions on its own task queue.
Workflows define the schedule (e.g., "fetch FRED data every hour", "check news every 5 min"). Temporal only handles scheduling, execution, and retries — the actual logic lives in normal service functions.

Key Temporal patterns:
- **Cron workflows** — for scheduled data fetching
- **Service functions** — normal Go functions that do the actual work (API calls, scraping, DB writes). Registered as Temporal activities but written as plain functions with no Temporal coupling.
- **Retry policies** — per-function (e.g., 3 retries, exponential backoff for rate limits)
- **Heartbeats** — for long-running scrape functions
- **Child workflows** — for parallel fan-out (e.g., one child per YouTube channel)

## 1.5 — Docker Compose

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

  nats:
    image: nats:latest
    command: ["--jetstream", "--store_dir", "/data"]
    volumes:
      - nats_data:/data
    ports:
      - "4222:4222"   # client
      - "8222:8222"   # monitoring

  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus_data:/prometheus
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana:latest
    environment:
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_ADMIN_PASSWORD:-admin}
    volumes:
      - grafana_data:/var/lib/grafana
    ports:
      - "3100:3000"
    depends_on:
      - prometheus
      - loki

  loki:
    image: grafana/loki:latest
    volumes:
      - loki_data:/loki
    ports:
      - "3200:3100"

  app:
    build: .
    env_file: .env
    environment:
      SERVICES_ENABLED: ""  # empty = all (servicepack built-in, no prefix)
      VOIDALPHA_DB_CONNECTIONSTRING: postgres://voidalpha:${POSTGRES_PASSWORD}@postgres:5432/voidalpha?sslmode=disable
      VOIDALPHA_TEMPORAL_ADDRESS: temporal:7233
      VOIDALPHA_NATS_URL: nats://nats:4222
    depends_on:
      - postgres
      - temporal
      - nats
    ports:
      - "3000:3000"   # api-gateway HTTP
      - "3001:3001"   # api-gateway WebSocket
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock  # for stealthy-auto-browse

volumes:
  pg_data:
  nats_data:
  prometheus_data:
  grafana_data:
  loki_data:
```

## 1.6 — Volume Backup & Restore

All persistent state lives in named Docker volumes. Temporal is stateless — its data lives in the `temporal` DB inside PostgreSQL, so backing up `pg_data` covers it.

**Volume inventory**:

| Volume | Service | Critical? | Notes |
|--------|---------|-----------|-------|
| `pg_data` | postgres | **YES** | All app data + Temporal workflow history. Loss = total data loss. |
| `nats_data` | nats | Low | JetStream message store. Transient notifications — can be rebuilt from DB state. |
| `prometheus_data` | prometheus | Low | Metrics history. Loss = lose historical graphs, no functional impact. |
| `grafana_data` | grafana | Medium | Dashboards and datasource config. Annoying to recreate manually. |
| `loki_data` | loki | Low | Log history. Loss = lose old logs, no functional impact. |

### PostgreSQL Backup (the critical one)

Use `pg_dump` — works while PG is running, no downtime.

```bash
# Backup both databases
COMPOSE_PROJECT=voidalpha
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR=./backups

mkdir -p $BACKUP_DIR

# App database (all the financial data)
docker compose exec -T postgres pg_dump -U voidalpha -Fc voidalpha \
  > $BACKUP_DIR/voidalpha_${TIMESTAMP}.dump

# Temporal database (workflow history)
docker compose exec -T postgres pg_dump -U voidalpha -Fc temporal \
  > $BACKUP_DIR/temporal_${TIMESTAMP}.dump
```

### PostgreSQL Restore

```bash
# Drop and recreate, then restore
docker compose exec -T postgres psql -U voidalpha -c "DROP DATABASE IF EXISTS voidalpha;"
docker compose exec -T postgres psql -U voidalpha -c "CREATE DATABASE voidalpha;"
docker compose exec -T postgres pg_restore -U voidalpha -d voidalpha \
  < $BACKUP_DIR/voidalpha_YYYYMMDD_HHMMSS.dump

# Same for temporal
docker compose exec -T postgres psql -U voidalpha -c "DROP DATABASE IF EXISTS temporal;"
docker compose exec -T postgres psql -U voidalpha -c "CREATE DATABASE temporal;"
docker compose exec -T postgres pg_restore -U voidalpha -d temporal \
  < $BACKUP_DIR/temporal_YYYYMMDD_HHMMSS.dump
```

### Generic Volume Backup (non-PG volumes)

Works for `nats_data`, `prometheus_data`, `grafana_data`, `loki_data`. Stop the service first to avoid corruption.

```bash
VOLUME=grafana_data
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

# Stop the service that uses the volume
docker compose stop grafana

# Backup
docker run --rm \
  -v ${COMPOSE_PROJECT}_${VOLUME}:/source:ro \
  -v $(pwd)/backups:/backup \
  alpine tar czf /backup/${VOLUME}_${TIMESTAMP}.tar.gz -C /source .

# Restart
docker compose start grafana
```

### Generic Volume Restore

```bash
VOLUME=grafana_data

# Stop the service
docker compose stop grafana

# Wipe and restore
docker run --rm \
  -v ${COMPOSE_PROJECT}_${VOLUME}:/target \
  -v $(pwd)/backups:/backup \
  alpine sh -c "rm -rf /target/* && tar xzf /backup/${VOLUME}_YYYYMMDD_HHMMSS.tar.gz -C /target"

# Restart
docker compose start grafana
```

### Automated Backup (cron on host)

```bash
# /etc/cron.d/voidalpha-backup — daily PG dump, weekly full volume backup
0 3 * * * cd /path/to/voidalpha && ./scripts/backup-pg.sh >> /var/log/voidalpha-backup.log 2>&1
0 4 * * 0 cd /path/to/voidalpha && ./scripts/backup-volumes.sh >> /var/log/voidalpha-backup.log 2>&1
```

Keep at least 7 daily PG dumps and 4 weekly volume backups. Rotate with `find $BACKUP_DIR -name "*.dump" -mtime +7 -delete`.

## 1.7 — Monitoring Stack

Prometheus scrapes metrics, Loki aggregates logs, Grafana visualizes both.

**Prometheus**:
- Scrapes app `/metrics` endpoint (standard Go `promhttp` handler)
- Also scrapes NATS monitoring (`nats:8222/metrics`), Temporal, and Postgres exporter if added later
- Config file: `prometheus.yml` in repo root, mounted read-only into the container

**Loki**:
- Log aggregation — services write to stdout, Docker log driver sends to Loki
- Alternative: Promtail sidecar scrapes container logs. Start with Docker log driver, switch if needed.
- Grafana queries Loki for log search/filtering

**Grafana**:
- Dashboards for: service health, NATS throughput, Temporal workflow success/failure, DB query latency, scrape success rates
- Pre-provisioned datasources: Prometheus + Loki (via provisioning YAML mounted into Grafana, or manual setup initially)
- Port 3100 on host → 3000 in container (avoids conflict with api-gateway on host port 3000)

**App metrics** (exposed at `/metrics` by api-gateway):
- `voidalpha_scrape_total{service, source, status}` — counter per scrape attempt
- `voidalpha_scrape_duration_seconds{service, source}` — histogram
- `voidalpha_events_published_total{subject}` — NATS publishes
- `voidalpha_ws_connections` — gauge of active WebSocket clients
- Custom per-service metrics as needed

## 1.8 — Shared Libraries (internal/pkg/)

Before writing services, set up shared packages. Note: `gonfiguration` (config parsing), `ctxerrors` (error wrapping), `goenv` (dev/prod), and `slog-configurator` (logging) are already provided by servicepack — no need to wrap them.

| Package | Purpose |
|---------|---------|
| `common` | General-purpose utilities shared across services — helpers, constants, common types, anything that doesn't belong to a specific domain package. Subpackages as needed (e.g. `common/httputil`, `common/timeutil`). |
| `database` | PostgreSQL connection pool (pgx), raw queries for now. GORM gen later. |
| `nats` | NATS JetStream client, stream/consumer setup, publish helpers |
| `temporal-client` | Shared Temporal client factory, common workflow options |
| `browser` | Go client for docker-stealthy-auto-browse HTTP API |
| `models` | Shared data types (MarketSnapshot, NewsArticle, etc.) |

## 1.9 — api-gateway Service

The first real service. Provides:

All endpoints are versioned per resource: `/api/v1/news`, `/api/v1/congress-trades`, etc. Each resource can bump its version independently without affecting others.

**WebSocket** (real-time, fed by NATS JetStream consumers):
- `WS /ws` — pushes events + snapshots to connected clients as they arrive from NATS

**REST APIs** (query dedicated tables directly):
- `GET /api/v1/snapshots?ticker=GC=F` — latest market snapshots
- `GET /api/v1/options?ticker=SPY` — options flow
- `GET /api/v1/gamma?ticker=SPY` — latest gamma exposure
- `GET /api/v1/earnings?ticker=AAPL` — earnings reports
- `GET /api/v1/news?limit=50` / `GET /api/v1/news/{id}` — news articles
- `GET /api/v1/youtube?limit=20` / `GET /api/v1/youtube/{id}` — YouTube transcripts
- `GET /api/v1/sentiment?indicator=fear_greed` — sentiment readings
- `GET /api/v1/macro?series=GDP` — macro indicators
- `GET /api/v1/fed/speeches` / `GET /api/v1/fed/meetings` / `GET /api/v1/fed/rates` — Fed data
- `GET /api/v1/econ-calendar` — economic calendar
- `GET /api/v1/congress-trades?ticker=NVDA` — congress trades
- `GET /api/v1/insider-trades?ticker=AAPL` — insider trades
- `GET /api/v1/short-interest?ticker=GME` — short interest
- `GET /api/v1/dark-pool?ticker=SPY` — dark pool volume
- `GET /api/v1/ftd?ticker=TSLA` — failure-to-deliver data

Frontend connects to the WebSocket and receives events as they happen.

## 1.10 — Frontend Scaffold

Svelte app compiled to static `index.html` + JS (no SSR, no Node server in production).

Initial pages (just shells for now, real implementation in phase 06):
- `/` — Dashboard (latest events across all categories)
- `/market` — Market data + snapshots
- `/news` — News feed
- `/options` — Options flow + gamma
- `/macro` — Macro + Fed data
- `/insider` — Insider + Congress trades
- `/sentiment` — Fear & Greed + social

Served by the api-gateway service (static file serving).

## 1.11 — Deliverables

After this phase we have:
- [ ] servicepack monorepo initialized, compiles, lints, tests pass
- [ ] PostgreSQL running with schema
- [ ] NATS JetStream running with EVENTS + SNAPSHOTS streams
- [ ] Temporal running with UI
- [ ] Prometheus scraping app `/metrics` + NATS metrics
- [ ] Grafana with Prometheus + Loki datasources configured
- [ ] Loki aggregating container logs
- [ ] Shared libraries (database, nats, temporal-client, browser, models, config)
- [ ] api-gateway service: REST API + WebSocket + `/metrics` endpoint + static file serving
- [ ] Svelte frontend shell: pages routing, WebSocket connection, renders incoming events
- [ ] Docker Compose runs everything with `docker compose up`
- [ ] Backup scripts: `scripts/backup-pg.sh`, `scripts/backup-volumes.sh` with rotation
- [ ] `.env.example` with all required variables documented
