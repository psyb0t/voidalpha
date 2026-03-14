# Future — Ideas & Optional Services

Things to build later, not part of the core phases.

---

## eToro Integration Service

Optional service that connects to the eToro public API when credentials are provided. Only activates when `ETORO_API_KEY` and `ETORO_USER_KEY` env vars are set.

**API Details:**
- Base URL: `https://public-api.etoro.com`
- Auth: `x-api-key` (public key) + `x-user-key` (generated user token) + `x-request-id` (UUID per request)
- WebSocket: `wss://ws.etoro.com/ws` (real-time price streaming)
- OpenAPI spec: `https://api-portal.etoro.com/api-reference/openapi.json` (v1.156.0)
- Full reference: `.research_files/etoro-api/ETORO_API_COMPLETE_REFERENCE.md`
- Test script: `etoro.py`

**What works (31 endpoints confirmed):**

Market data:
- Live bid/ask rates for 15,427 instruments (stocks, forex, crypto, commodities, indices)
- OHLCV candles: 1min/5min/15min/30min/1hr/4hr/daily/weekly, up to 1000 candles
- Closing prices (daily/weekly/monthly) for all instruments
- Search instruments by name/ticker
- Industry/sector classification
- Instrument metadata with price changes (daily/weekly/monthly/3mo/6mo/1yr)

Crowd sentiment (unique — can't get this from Finnhub/FRED):
- `popularityUniques` / `popularityUniques7Day/14Day/30Day` — total unique users interested
- `holdingPct` / `buyHoldingPct` / `sellHoldingPct` — % of users holding long vs short
- `buyPctChange24Hours` — shift in buy/sell ratio over 24h
- `traders7DayChange/14DayChange/30DayChange` — trader activity trends
- People search (3.2M users) filtered by risk score, gain, instrument holdings

Social:
- Discussion feeds per instrument (AAPL, BTC, etc.)
- Curated lists ("Trending on News", etc.)
- User performance data (gain history, risk scores, copier stats)

Personal trading (if user wants to monitor their eToro account):
- Portfolio positions, PnL, trade history
- WebSocket for real-time price + portfolio notifications

**What doesn't work:**
- Market recommendations (204 empty)
- Demo endpoints (need demo user key — we have a separate demo key)
- Sub-portfolios, copiers (need higher permissions / Popular Investor status)
- Some user info endpoints (tradeinfo, live portfolio — may need correct username format)

**Potential uses in VoidAlpha:**
- Secondary/fallback market data source (quotes, candles)
- Crowd sentiment signals (holding ratios, popularity trends) → feed into sentiment-tracker
- Social feed analysis → feed into news-aggregator or categorizer
- Personal portfolio monitoring dashboard page
- eToro "trending" instruments as a sentiment signal

**Keys (two separate user keys — demo and real):**
- API key (public, shared): in `etoro.py`
- Demo user key: in `etoro.py` as `DEMO_USER_KEY`
- Real user key: in `etoro.py` as `REAL_USER_KEY`

---

## Saxo Bank Integration (Options Trading + Algo)

Optional service for automated options trading via Saxo Bank OpenAPI. Only broker with a proper API that supports options from Romania/EU. Activates when `SAXO_API_KEY` env var is set.

**Why Saxo:**
- Only real alternative to Interactive Brokers for options API in EU
- eToro has no options (US-only), DEGIRO has no official API (will ban you for automating)
- Free account, no inactivity fee, no platform fee
- Custody fee 0.15%/year waivable via stock lending program (specifically for Central/Eastern Europe)

**API Details:**
- Developer portal: https://www.developer.saxo/
- REST API with streaming support
- Python SDK: `saxo-openapi` on PyPI (https://github.com/hootnot/saxo_openapi)
- 3,100+ listed options across equities, energy, metals from 20 exchanges
- 30,000+ total instruments

**Potential uses in VoidAlpha:**
- Automated options trading (open/close positions, limit orders)
- Options chain data as additional source for options-flow service
- Portfolio management and position monitoring
- Real-time streaming for options prices
- Algo trading strategies based on VoidAlpha signals (GEX, sentiment, macro events)

**TODO:**
- Sign up for Saxo account
- Get API credentials from developer portal
- Research full OpenAPI spec (clone docs into .research_files/saxo-api/)
- Write `saxo.py` test script (same pattern as etoro.py/caca.py)
- Check what options chain data is available (Greeks, chains, expirations)
- Check if they have options flow / unusual activity data
