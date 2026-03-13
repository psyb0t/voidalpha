# 00 — VoidAlpha: Architecture Overview

## What This Is

A real-time financial intelligence platform built as a servicepack monorepo.
Multiple microservices, each dedicated to one data niche, feed a central PostgreSQL
database. A Go backend pushes updates to a Svelte frontend via WebSocket, triggered
by PostgreSQL LISTEN/NOTIFY. Everything runs in Docker Compose. Aimed at swing traders
and longer-term investors.

## Tech Stack

| Layer | Tech |
|-------|------|
| Monorepo framework | [servicepack](https://github.com/psyb0t/servicepack) |
| Backend language | Go |
| Frontend | Svelte → compiled to `index.html` + pure JS |
| Database | PostgreSQL (LISTEN/NOTIFY for real-time events). Two DBs: `voidalpha` (app data) + `temporal` (workflow engine). Raw SQL + pgx first, GORM gen later. |
| Workflow engine | Temporal (scheduling, retries, orchestration) |
| Web scraping | [docker-stealthy-auto-browse](https://github.com/psyb0t/docker-stealthy-auto-browse) on demand, direct HTTP + goquery when possible, RSS feeds |
| AI processing | [docker-claude-code](https://github.com/psyb0t/docker-claude-code) programmatic mode, Haiku model. Handles all summarization, categorization, urgency, sentiment, hawkish/dovish scoring. |
| Secrets | `.env` file mounted into containers |
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
Tagging assigns correct regions regardless (US, EU, Asia, etc.) so filtering works when we expand.

EU and Asia data source expansion comes later.

## Service Map

Each service is a servicepack `internal/pkg/services/<name>/` service.
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
| `categorizer` | **All AI processing**: summarization, tagging, urgency, sentiment, hawkish/dovish scoring, cross-service correlation | Claude Haiku via docker-claude-code |
| `api-gateway` | HTTP API + WebSocket server for frontend | Internal |

## Data Flow

```
[Data Source APIs / RSS / Scrapers]
        ↓
[Temporal Workflow → Activity]
        ↓
[Service writes RAW event to PostgreSQL]
        ↓
[PostgreSQL trigger → NOTIFY on channel]
        ↓ (two consumers)
        ├→ [api-gateway LISTEN → push to WebSocket → Svelte frontend]
        └→ [categorizer LISTEN → fetch from export API → JSON files → docker-claude-code (Haiku) → read results → update DB]
                ↓
        [categorizer sweep (every 5 min) catches any untagged events via export API]
```

## Category / Tagging System

Every data point gets tagged by the categorizer (Claude Haiku):
- **Asset class**: equity, commodity, crypto, forex, bond
- **Region**: US, EU, Asia — inferred from source metadata when available, Claude for the rest
- **Sector**: technology, energy, defense, healthcare, finance, etc.
- **Ticker(s)**: specific symbols when applicable (AAPL, GC=F, ^GSPC)
- **Category**: news, macro, insider, options, sentiment, earnings, political
- **Urgency**: routine, notable, alert
- **Sentiment**: bullish, bearish, neutral
- **Summary**: AI-generated 2-3 sentence summary

This lets the frontend filter and correlate across data types.

## Key Decisions

1. **servicepack monorepo** — one binary, all services, `SERVICES_ENABLED` env var to filter
2. **Scraping hierarchy** — direct HTTP + goquery first, RSS feeds, stealthy-auto-browse only when JS rendering is required
3. **Temporal for orchestration** — no hand-rolled cron, no manual retries
4. **PostgreSQL NOTIFY** — no separate message queue needed for frontend updates
5. **Separate DBs** — `voidalpha` for app data, `temporal` for workflow engine, same Postgres instance
6. **Categorizer owns all AI** — data services write raw events, categorizer does all Claude Haiku processing (summarization, tagging, urgency, sentiment, scoring). No inline AI in data services.
7. **No fallbacks for AI** — if Claude fails, Temporal retries. Untagged events sit until categorizer sweep picks them up.
8. **Raw SQL first** — pgx with hand-written queries. GORM gen comes later.
9. **.env file** — simple secret management, good enough for self-hosted

## Phases

| Phase | Name | What |
|-------|------|------|
| 01 | Foundation | servicepack init, PostgreSQL, Temporal, docker-compose, shared libs, API gateway, frontend scaffold |
| 02 | Market Data | real-time quotes, OHLCV, options chains, gamma, earnings |
| 03 | News & Sentiment | news aggregation (financial + world), fear & greed, social sentiment |
| 04 | Macro & Fed | FRED, BLS, Fed speeches, economic calendar |
| 05 | Institutional & Political | Congress trades, insider trading, FINRA/dark pool |
| 06 | Categorization & Linking | AI-powered tagging, summarization, cross-referencing via Claude Haiku |
| 07 | Frontend | Svelte dashboard, real-time pages, filtering, search |
