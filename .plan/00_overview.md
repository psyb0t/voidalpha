# 00 — VoidAlpha: Architecture Overview

## What This Is

A real-time financial intelligence platform built as a servicepack monorepo.
Multiple microservices, each dedicated to one data niche, feed a central PostgreSQL
database. A Go backend pushes updates to a Svelte frontend via WebSocket, fed
by NATS JetStream. Everything runs in Docker Compose. Aimed at swing traders
and longer-term investors.

## Tech Stack

| Layer | Tech |
|-------|------|
| Monorepo framework | [servicepack](https://github.com/psyb0t/servicepack) |
| Backend language | Go 1.25, `log/slog` logging, Docker-based static build |
| Frontend | Svelte → compiled to `index.html` + pure JS |
| Database | PostgreSQL. Two DBs: `voidalpha` (app data) + `temporal` (workflow engine). Raw SQL + pgx first, GORM gen later. |
| Event bus | NATS JetStream — at-least-once delivery per consumer group, replaces PG LISTEN/NOTIFY |
| Workflow engine | Temporal (scheduling, retries, orchestration) |
| Web scraping | [docker-stealthy-auto-browse](https://github.com/psyb0t/docker-stealthy-auto-browse) on demand, direct HTTP + goquery when possible, RSS feeds |
| Secrets | `.env` file mounted into containers |
| Monitoring | Prometheus (metrics) + Grafana (dashboards) + Loki (log aggregation) |
| Infra | Docker Compose, single machine (services can run on separate machines, shared DB) |

## Scope: US-First

Phase 1 data sources are US-focused:
- US equities, indices (S&P 500, Nasdaq, Dow, Russell 2000)
- Commodities (Gold, Silver, Oil, Nat Gas)
- Crypto (BTC, ETH)
- US macro (FRED, BLS, Fed, Treasury)
- US political (Congress trades, insider trading)
- US options flow, gamma, dark pools

News ingestion is global — geopolitical, tech, energy, regulatory events all move markets.

EU and Asia data source expansion comes later.

## Service Map

Each service is a servicepack `internal/pkg/services/<name>/` service (`make service NAME=x`).
Each implements `Service` interface (`Name()`, `Run(ctx)`, `Stop(ctx)`) with `New() (*T, error)` constructor.
Optional: `Retryable`, `AllowedFailure`, `Dependent` interfaces for retry/deps/graceful degradation.
Each starts a Temporal worker internally for its own workflows.

| Service | Data Niche | Sources |
|---------|-----------|---------|
| `market-data` | Real-time/delayed quotes, OHLCV | Finnhub, Yahoo Finance, Massive (ex-Polygon.io) |
| `options-flow` | Options chains, gamma exposure, unusual activity | CBOE delayed JSON (free, primary), Tradier, Massive |
| `news-aggregator` | All news — financial, geopolitical, tech, regulatory | 18+ RSS feeds (financial + world), Finnhub News, NewsAPI |
| `sentiment-tracker` | Fear & Greed, put/call ratio, VIX signals, AAII | CNN F&G, CBOE CSVs, AAII, Reddit |
| `macro-data` | Economic indicators, GDP, CPI, jobs, yields | FRED API, BLS, Treasury.gov |
| `fed-watcher` | Fed speeches, FOMC minutes, rate decisions | Fed RSS, FOMC calendar, CME FedWatch |
| `congress-trades` | Congress member stock trades | Capitol Trades, Quiver Quant |
| `insider-trading` | Corporate insider buy/sell filings | SEC EDGAR, Finnhub, OpenInsider |
| `finra-data` | Short interest, dark pool ATS volume | FINRA Query API, FINRA CDN |
| `earnings-calendar` | Upcoming/recent earnings, EPS estimates | Finnhub, Yahoo Finance |
| `econ-calendar` | Upcoming economic releases (CPI, NFP, FOMC, GDP) | Finnhub, FRED releases |
| `youtube-intel` | YouTube analyst transcripts — macro, trading, quant analysis | yt-dlp transcripts from configurable channel list |
| `api-gateway` | HTTP API + WebSocket server for frontend | Internal |

## Data Flow

```
[Data Source APIs / RSS / Scrapers]
        ↓
[Temporal Workflow → Service Function]
        ↓
[Service writes to DEDICATED TABLE (news_articles, congress_trades, etc.)]
        ↓
[Service publishes to NATS JetStream (subject: events.{table_name})]
        ↓
[Each durable consumer group gets the message at least once]
        ↓
[api-gateway consumer → push to WebSocket → Svelte frontend]

[market_snapshots → NATS subject snapshots.{ticker} → WebSocket → real-time ticker bar]
```

## Key Decisions

1. **servicepack monorepo** — one binary, all services, `SERVICES_ENABLED` env var to filter
2. **Scraping hierarchy** — direct HTTP + goquery first, RSS feeds, stealthy-auto-browse only when JS rendering is required
3. **Temporal for orchestration** — no hand-rolled cron, no manual retries
4. **NATS JetStream** — at-least-once delivery per consumer group, durable consumers survive restarts
5. **Separate DBs** — `voidalpha` for app data, `temporal` for workflow engine, same Postgres instance
6. **Raw SQL first** — pgx with hand-written queries. GORM gen comes later.
    - **UUID primary keys** on all tables (`gen_random_uuid()`). No BIGSERIAL.
    - **Dedicated tables per data type** — each service writes structured data to its own table (news_articles, congress_trades, etc.). No shared notification table — NATS handles event delivery.
7. **.env file** — simple secret management, good enough for self-hosted
8. **Per-resource API versioning** — `/api/v1/news`, `/api/v1/congress-trades`, etc. Each resource bumps its version independently.
9. **Server time is UTC, always TIMESTAMPTZ** — all backend timestamps stored and served in UTC. All DB columns use `TIMESTAMPTZ` (not `TIMESTAMP`) so timezone info survives migrations. Timezone conversion is strictly a frontend concern (auto-detected from browser or user-selectable).
10. **Code style follows servicepack conventions** — all code must use the same libraries and patterns as servicepack:
    - **Errors**: `ctxerrors.Wrap(err, "context")` / `ctxerrors.Wrapf(err, "context %s", val)` — never raw `fmt.Errorf` or naked error returns. Every error gets wrapped with context. Every predictable error condition must be a sentinel (`var ErrSomething = errors.New("something")`) defined in `errors.go` within its package. Check with `errors.Is()`. `errors.New` is ONLY for sentinel definitions at package level — never inline in a return. To return an error, either return a sentinel directly or wrap one with `ctxerrors.Wrap(ErrSomething, "extra context")`.
    - **Logging**: `log/slog` only (`slog.Info`, `slog.Error`, `slog.Warn`, `slog.Debug`) with structured key-value pairs. No logrus, no zerolog, no fmt.Println.
    - **Config**: `gonfiguration.Parse(&cfg)` with struct tags `env:"VAR_NAME"`. Defaults via `gonfiguration.SetDefaults()`. No manual `os.Getenv`.
    - **Env var naming**: `VOIDALPHA_SERVICENAME_PACKAGENAME_PROPERTYNAME` — underscores delimit app → service → package → field. Examples: `VOIDALPHA_APIGATEWAY_HTTP_LISTENADDRESS`, `VOIDALPHA_MARKETDATA_DB_CONNECTIONSTRING`, `VOIDALPHA_NEWSSAGGREGATOR_FINNHUB_APIKEY`. Shared/app-level vars drop the service: `VOIDALPHA_DB_CONNECTIONSTRING`, `VOIDALPHA_NATS_URL`.
    - **Environment**: `goenv.Get()` for dev/prod detection. No hand-rolled env checks.
    - **Service pattern**: `ServiceName` const, `New() (*T, error)` constructor, implement `Service` interface. Optional `Retryable`, `AllowedFailure`, `Dependent`.
    - **Return early**, no if-else chains. `continue` in loops, `break` in selects/switches.
    - **No stdlib duplication** — if servicepack already provides a capability (config, errors, logging, env), use the servicepack way. Don't import competing libraries.

## Phases

| Phase | Name | What |
|-------|------|------|
| 01 | Foundation | servicepack init, PostgreSQL, Temporal, docker-compose, shared libs, API gateway, frontend scaffold |
| 02 | Market Data | real-time quotes, OHLCV, options chains, gamma, earnings |
| 03 | News & Sentiment | news aggregation (financial + world), fear & greed, social sentiment |
| 04 | Macro & Fed | FRED, BLS, Fed speeches, economic calendar |
| 05 | Institutional & Political | Congress trades, insider trading, FINRA/dark pool |
| 06 | Frontend | Svelte dashboard, real-time pages, filtering, search |
