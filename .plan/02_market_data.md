# 02 — Market Data & Options

Real-time/delayed quotes, OHLCV candles, options chains, gamma exposure, unusual activity, and earnings calendar.

## Services in This Phase

| Service | Task Queue | Schedule |
|---------|-----------|----------|
| `market-data` | `market-data-queue` | Every 1–5 min (quotes), every 15 min (OHLCV) |
| `options-flow` | `options-flow-queue` | Every 15 min (chains), every 30 min (gamma calc) |
| `earnings-calendar` | `earnings-calendar-queue` | Every 6 hours |

---

## 2.1 — market-data Service

### What It Does
Fetches real-time/delayed stock quotes, index values, commodity prices, and crypto prices. Writes to `market_snapshots` table.

### Tracked Assets (Phase 1)

**Indices**: `^GSPC` (S&P 500), `^IXIC` (Nasdaq), `^DJI` (Dow), `^RUT` (Russell 2000), `^VIX` (VIX)
**Commodities**: `GC=F` (Gold), `SI=F` (Silver), `CL=F` (WTI Crude), `NG=F` (Nat Gas)
**Crypto**: `BTC-USD`, `ETH-USD`
**Top equities**: Configurable watchlist (default: top 50 S&P 500 by market cap)

### Data Sources

#### Primary: Finnhub
- **Base URL**: `https://finnhub.io/api/v1`
- **Auth**: `?token=` query param or `X-Finnhub-Token` header
- **Go SDK**: `github.com/Finnhub-Stock-API/finnhub-go/v2` (official, OpenAPI-generated)
- **Key Endpoints**:
  - `GET /quote?symbol=AAPL` — real-time quote (c=price, d=change, dp=change%, h, l, o, pc)
  - `GET /stock/candle?symbol=AAPL&resolution=D&from=X&to=Y` — OHLCV candles
  - `GET /crypto/candle?symbol=BINANCE:BTCUSDT&resolution=D&from=X&to=Y` — crypto candles
- **Free Tier**: 60 calls/min, 30 calls/sec hard cap. No daily limit. US stocks + crypto + forex.
- **Env Var**: `FINNHUB_API_KEY`
- **Notes**: No options chain data at any tier. Use for quotes, candles, news, insider, earnings only.

#### Secondary: Massive (formerly Polygon.io, rebranded Oct 2025)
- **Base URL**: `https://api.massive.com` (legacy `api.polygon.io` still works)
- **Auth**: `Authorization: Bearer` header
- **Go SDK**: `github.com/massive-com/client-go/v3` (official, Go 1.21+, generics, auto-pagination)
- **Key Endpoints**:
  - `GET /v2/aggs/ticker/{ticker}/prev` — previous day OHLCV
  - `GET /v2/aggs/ticker/{ticker}/range/{multiplier}/{timespan}/{from}/{to}` — historical aggregates
  - `GET /v2/snapshot/locale/us/markets/stocks/tickers/{ticker}` — ticker snapshot
  - `GET /v3/reference/tickers` — ticker details
- **Free Tier**: 5 calls/min, delayed 15 min, end-of-day data only
- **Paid**: Starter $29 (delayed), Developer $79 (real-time), Advanced $199 (real-time + WebSocket + options)
- **Env Var**: `MASSIVE_API_KEY`

#### Fallback: Yahoo Finance (unofficial)
- **Base URL**: `https://query1.finance.yahoo.com` (load-balanced with `query2`)
- **Auth**: Crumb + session cookie required. Fetch page → extract `CrumbStore` → carry cookies + `?crumb=` on all requests.
- **Go Library**: `github.com/oscarli916/yahoo-finance-api` (handles crumb auth, updated Feb 2026)
- **Key Endpoints**:
  - `GET /v8/finance/chart/{symbol}?range=1d&interval=5m` — OHLCV chart data
  - `GET /v7/finance/options/{symbol}` — options chain (no Greeks, just price/vol/OI/IV)
  - `GET /v10/finance/quoteSummary/{symbol}?modules=financialData`
- **Notes**: No official API, no SLA, may break. Use stealthy-auto-browse as last resort if crumb auth breaks.
- **No API key needed**

### Temporal Workflows

```
QuoteFetchWorkflow (cron: every 2 min)
  → FetchQuotesActivity(tickers []string) → []Quote
  → WriteSnapshotsActivity(quotes []Quote)

CandleFetchWorkflow (cron: every 15 min)
  → FetchCandlesActivity(ticker, resolution, from, to) → []Candle
  → WriteCandlesActivity(candles []Candle)
```

**Retry Policy**: 3 attempts, initial interval 5s, backoff coefficient 2.0, max interval 30s.

### Config

```go
type Config struct {
    FinnhubAPIKey  string `env:"FINNHUB_API_KEY"`
    MassiveAPIKey  string `env:"MASSIVE_API_KEY"`
    WatchlistFile  string `env:"WATCHLIST_FILE" default:""`  // optional override
    QuoteInterval  string `env:"MARKET_QUOTE_INTERVAL" default:"2m"`
    CandleInterval string `env:"MARKET_CANDLE_INTERVAL" default:"15m"`
}
```

### Data Written

Table: `market_snapshots`
- `ticker`, `price`, `change_pct`, `volume`
- `extra` JSONB: `{"open": X, "high": X, "low": X, "prev_close": X, "bid": X, "ask": X, "source": "finnhub"}`

Also writes to `data_events` for significant moves (>2% change triggers `urgency: notable`, >5% triggers `urgency: alert`).

---

## 2.2 — options-flow Service

### What It Does
Fetches options chains, computes gamma exposure (GEX), tracks unusual options activity. Writes to `options_flow` and `gamma_exposure` tables.

### Tracked Tickers
`SPY`, `QQQ`, `IWM`, `AAPL`, `MSFT`, `NVDA`, `TSLA`, `AMZN`, `META`, `GOOG` — configurable via env.

### Data Sources

#### Primary: CBOE Delayed Options JSON (FREE, no auth)
- **URL**: `https://cdn.cboe.com/api/global/delayed_quotes/options/{TICKER}.json`
- **Index tickers**: Use underscore prefix: `_SPX`, `_VIX`, `_NDX`
- **Auth**: None. Free, no API key, no rate limit documented.
- **Delay**: 15 minutes
- **Returns**: Full options chain with **all Greeks** (gamma, delta, theta, vega), strike, bid, ask, volume, open_interest, IV, expiration. Also returns `current_price` (spot price for GEX calc).
- **Notes**: This is the primary free source for GEX computation. No account needed. Reference implementation in `.research_files/gex_data/cboe_data.py`.

#### Secondary: Tradier
- **Base URL**: `https://api.tradier.com/v1` (production) / `https://sandbox.tradier.com/v1` (sandbox)
- **Auth**: `Authorization: Bearer {token}` + `Accept: application/json`
- **Go Libraries**: `github.com/timpalpant/go-tradier` (LGPL-3.0), `github.com/gnagel/go-tradier` (fork with retries)
- **Key Endpoints**:
  - `GET /markets/options/chains?symbol=SPY&expiration=2026-03-20&greeks=true` — full chain with ORATS Greeks
  - `GET /markets/options/expirations?symbol=SPY` — available expiration dates
  - `GET /markets/options/strikes?symbol=SPY&expiration=2026-03-20` — strike prices
  - `GET /markets/quotes?symbols=SPY` — underlying quote for spot price
- **Sandbox**: Free, 60 req/min, 15-min delayed data. No account needed.
- **Production**: Requires brokerage account. 120 req/min. Lite ($0/mo + commissions), Pro ($10/mo, commission-free), Pro Plus ($35/mo).
- **Env Var**: `TRADIER_API_KEY`
- **Notes**: Better for equities options (wider ticker coverage than CBOE CDN). Use as supplement/fallback to CBOE.

#### Tertiary: Massive (formerly Polygon.io) Options
- **Base URL**: `https://api.massive.com`
- **Key Endpoints**:
  - `GET /v3/snapshot/options/{underlyingAsset}` — options chain snapshot (with strike/expiry filters)
  - `GET /v3/reference/options/contracts/{optionsTicker}` — contract reference
- **Requires**: Advanced plan ($199/mo) for options chain data
- **Env Var**: `MASSIVE_API_KEY` (shared with market-data)
- **Notes**: Only worth it if we need real-time (not delayed) options data. Skip for MVP.

### Gamma Exposure Calculation

GEX is computed locally from the options chain. Reference implementation in `.research_files/gex-tracker/main.py` and `.research_files/gex_data/gamma_exposure.py`.

```
For each option contract:
  contract_gex = gamma × OI × 100 × spot² × 0.01

  If call: gex = +contract_gex (dealers long gamma = stabilizing)
  If put:  gex = -contract_gex (dealers short gamma = destabilizing)

Total GEX = sum of all contract_gex
  Normalize: total_gex_billions = total_gex / 1e9

Max Pain = strike with highest total combined OI
```

**Gamma Flip Point**: The spot price where total net GEX crosses zero. Below this, dealers amplify moves (short gamma). Above, dealers suppress volatility (long gamma). Computed by scanning GEX across hypothetical spot prices and finding the zero crossing.

**Practical interpretation**:
- **Positive GEX**: Dealers buy dips, sell rips. Low volatility, mean-reversion.
- **Negative GEX**: Dealers amplify moves. High volatility, trends accelerate.
- **High OI at strike near expiry**: Pin risk to that strike.
- **0DTE options**: Extremely high gamma, can dominate total GEX on expiration days.

If gamma is not provided by the data source (e.g., Yahoo Finance), it can be computed via Black-Scholes from implied volatility + time to expiry.

### Temporal Workflows

```
OptionsChainWorkflow (cron: every 15 min, market hours only)
  → FetchExpirationsActivity(ticker) → []date
  → FetchChainActivity(ticker, expiration) → []OptionContract
  → WriteOptionsFlowActivity(contracts)
  → ComputeGammaActivity(ticker, contracts, spotPrice) → GammaExposure
  → WriteGammaActivity(gex)

UnusualActivityWorkflow (cron: every 30 min, market hours only)
  → ScrapeUnusualActivity() → []UnusualOption
  → WriteDataEventsActivity(events)  // category: 'options', urgency based on volume
```

**Market hours check**: Skip execution if outside 9:30 AM – 4:00 PM ET on weekdays.

### Config

```go
type Config struct {
    TradierAPIKey    string `env:"TRADIER_API_KEY"`
    PolygonAPIKey    string `env:"POLYGON_API_KEY"`
    OptionsTickers   string `env:"OPTIONS_TICKERS" default:"SPY,QQQ,IWM,AAPL,MSFT,NVDA,TSLA,AMZN,META,GOOG"`
    ChainInterval    string `env:"OPTIONS_CHAIN_INTERVAL" default:"15m"`
    MarketTZ         string `env:"MARKET_TIMEZONE" default:"America/New_York"`
}
```

### Data Written

Table: `options_flow`
- `ticker`, `expiry`, `strike`, `option_type`, `volume`, `open_interest`, `implied_vol`, `delta`, `gamma`
- `extra` JSONB: `{"theta": X, "vega": X, "bid": X, "ask": X, "last": X, "source": "tradier"}`

Table: `gamma_exposure`
- `ticker`, `spot_price`, `total_gex`, `call_gex`, `put_gex`, `max_pain`
- `gex_by_strike` JSONB: `[{"strike": 500, "gex": 1234567}, ...]`

---

## 2.3 — earnings-calendar Service

### What It Does
Tracks upcoming and recent earnings reports, EPS estimates vs actuals. Writes to `data_events` with `category: 'earnings'`.

### Data Sources

#### Primary: Finnhub
- **Endpoint**: `GET /calendar/earnings?from=2026-03-01&to=2026-03-31`
- **Returns**: `{earningsCalendar: [{date, epsActual, epsEstimate, hour, quarter, revenueActual, revenueEstimate, symbol, year}]}`
- **Free Tier**: Included in free plan
- **Env Var**: `FINNHUB_API_KEY` (shared)

#### Secondary: Yahoo Finance
- Scrape earnings calendar page as fallback
- Use stealthy-auto-browse if needed

### Temporal Workflows

```
EarningsCalendarWorkflow (cron: every 6 hours)
  → FetchEarningsActivity(from, to) → []EarningsReport
  → DeduplicateActivity(reports) → []EarningsReport  // check source_id
  → WriteDataEventsActivity(reports)

EarningsSurpriseWorkflow (cron: every 1 hour during earnings season)
  → FetchRecentEarningsActivity() → []EarningsReport
  → CheckSurprisesActivity(reports) → []EarningsReport  // flag big misses/beats
  → WriteDataEventsActivity(surprises)  // urgency: 'notable' for >10% surprise, 'alert' for >20%
```

### Data Written

Table: `data_events`
- `source_service`: `'earnings-calendar'`
- `category`: `'earnings'`
- `title`: `"AAPL Q1 2026 Earnings: Beat by $0.05"` or `"AAPL Earnings Due 2026-03-15 AMC"`
- `body` JSONB: `{"symbol": "AAPL", "date": "2026-03-15", "hour": "amc", "eps_estimate": 1.50, "eps_actual": 1.55, "revenue_estimate": 95000000000, "revenue_actual": 96000000000, "surprise_pct": 3.33}`
- `urgency`: `routine` for upcoming, `notable` for >10% surprise, `alert` for >20%
- `source_id`: `"finnhub-earnings-AAPL-2026-Q1"`

---

## 2.4 — Deliverables

After this phase:
- [ ] `market-data` service fetching quotes + candles from Finnhub/Polygon, writing to `market_snapshots`
- [ ] `options-flow` service fetching chains from Tradier, computing GEX, writing to `options_flow` + `gamma_exposure`
- [ ] `earnings-calendar` service tracking earnings dates + surprises
- [ ] All three services running with Temporal cron workflows
- [ ] Significant market moves auto-generate `data_events` with appropriate urgency
- [ ] WebSocket pushes snapshot + options + earnings events to frontend
