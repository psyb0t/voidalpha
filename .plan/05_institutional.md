# 05 — Institutional & Political

Congress member stock trades, corporate insider buy/sell, FINRA short interest, dark pool volume.

## Services in This Phase

| Service | Task Queue | Schedule |
|---------|-----------|----------|
| `congress-trades` | `congress-trades-queue` | Every 2 hours |
| `insider-trading` | `insider-trading-queue` | Every 1 hour |
| `finra-data` | `finra-data-queue` | Every 4–12 hours |

---

## 5.1 — congress-trades Service

### What It Does
Tracks stock trades by US Congress members (STOCK Act disclosures). Detects large trades, cluster buying/selling, and trades in sectors where the member has committee oversight.

### Data Sources

#### Primary: Capitol Trades
- **URL**: `https://www.capitoltrades.com/trades`
- **Method**: Scrape with stealthy-auto-browse — JS-rendered React app
- **Data**: Congress member name, party, chamber, ticker, trade type (buy/sell), amount range, filed date, trade date
- **Schedule**: Every 2 hours
- **Scrape Flow**:
  1. `goto` → `https://www.capitoltrades.com/trades?page=1&pageSize=50`
  2. `get_text` → parse trade table
  3. Paginate if needed (check for new trades since last scrape)
- **No API key needed**

#### Secondary: Quiver Quant
- **URL**: `https://www.quiverquant.com/congresstrading/`
- **Method**: Scrape public dashboard with stealthy-auto-browse (free dashboard, no login needed)
- **API**: `https://api.quiverquant.com/` — paid only, starts at $10/mo (Hobbyist). Covers congress trades, insider trades, lobbying, gov contracts, Reddit WSB mentions.
- **Data**: Similar to Capitol Trades, cross-reference for completeness
- **Schedule**: Every 4 hours (lower priority, used for gap-filling)
- **Notes**: Scrape the public page for now. Consider API subscription later if scraping becomes unreliable.

#### House/Senate Disclosure Sites (official)
- **House**: `https://disclosures-clerk.house.gov/FinancialDisclosure`
- **Senate**: `https://efdsearch.senate.gov/search/`
- **Method**: Scrape with stealthy-auto-browse. These are the primary source but less user-friendly than aggregators.
- **Schedule**: Every 6 hours (slower, official source for verification)

### Deduplication

Dedup via `source_id UNIQUE` constraint on the `congress_trades` table.

Format: `congress-{member_slug}-{ticker}-{trade_date}-{amount_range}`

Example: `congress-nancy-pelosi-NVDA-2026-03-01-1M-5M`

### Temporal Workflows

```
CongressTradesWorkflow (cron: every 2 hours)
  → ScrapeCapitolTrades(lastScrapeTime) → []CongTrade
  → Deduplicate(trades) → []CongTrade
  → Enrich(trades) → []CongTrade  // add committee info, sector context
  → ClassifyUrgency(trades) → []CongTrade
  → WriteCongressTrades(trades)
  → PublishNATS(trades)  // publish to events.congress_trades

QuiverBackfillWorkflow (cron: every 4 hours)
  → ScrapeQuiver() → []CongTrade
  → CrossReference(trades) → []CongTrade  // only keep ones not in Capitol Trades
  → WriteCongressTrades(trades)
  → PublishNATS(trades)  // publish to events.congress_trades
```

### Urgency Classification

- **alert**: Trade >$1M, or multiple members trading same ticker within 48h (cluster signal)
- **notable**: Trade >$100K, or member sits on committee overseeing the sector
- **routine**: Regular small trades

### Config

```go
type Config struct {
    BrowserAPIURL      string `env:"BROWSER_API_URL" default:"http://localhost:8080"`
    FetchInterval      string `env:"CONGRESS_FETCH_INTERVAL" default:"2h"`
}
```

### Data Written

**Primary table: `congress_trades`** (UUID primary key)
- `member`, `party`, `chamber`, `ticker`, `trade_type`, `amount_range`, `trade_date`, `filed_date`, `committees` (JSONB), `source`, `source_id` (UNIQUE)

After each write, publishes to NATS JetStream subject `events.congress_trades` with `{"id": "uuid", "table": "congress_trades"}` payload.

---

## 5.2 — insider-trading Service

### What It Does
Tracks corporate insider (CEO, CFO, directors, 10%+ owners) stock transactions from SEC Form 4 filings.

### Data Sources

#### Primary: SEC EDGAR
- **Auth**: None, but **must** set `User-Agent: "VoidAlpha admin@voidalpha.com"` header (SEC requirement)
- **Rate Limit**: 10 requests/sec (hard limit, IP block if exceeded)
- **Key Endpoints**:
  - `https://efts.sec.gov/LATEST/search-index?forms=4&dateRange=custom&startdt=2026-03-01&enddt=2026-03-12` — full-text search for Form 4 filings
  - `https://data.sec.gov/submissions/CIK{10-digit}.json` — company submissions index
  - `https://data.sec.gov/api/xbrl/companyfacts/CIK{10-digit}.json` — structured XBRL data
- **Bulk Data**: Quarterly index files at `https://www.sec.gov/Archives/edgar/full-index/YYYY/QN/form.idx`
- **No API key needed**

#### Secondary: Finnhub Insider Transactions
- **Endpoint**: `GET /stock/insider-transactions?symbol=AAPL`
- **Returns**: `[{name, share, change, filingDate, transactionDate, transactionCode, transactionPrice}]`
- **Transaction Codes**: `P` = purchase, `S` = sale, `A` = grant/award, `M` = exercise
- **Free Tier**: Included
- **Env Var**: `FINNHUB_API_KEY` (shared)

#### Tertiary: OpenInsider
- **URL**: `http://openinsider.com/screener?s={TICKER}&xp=1&xs=1&fd=-1&cnt=100&action=1`
- **Method**: Basic HTTP scraping of HTML tables — no stealth browser needed, not heavily JS-dependent
- **Pre-built Screeners**: `/latest-cluster-buys`, `/latest-insider-activity`, `/{TICKER}`
- **Data**: Filing date, trade date, ticker, insider name/title, transaction type, price, quantity, value, shares owned, post-trade returns (1d/1w/1m/6m)
- **Schedule**: Every 2 hours
- **Notes**: Underlying data is SEC Form 4. OpenInsider adds enrichment like post-trade return calculations. Good for cluster buy/sell detection.

### Temporal Workflows

```
InsiderFilingsWorkflow (cron: every 1 hour)
  → FetchFinnhubInsider(tickers) → []InsiderTrade
  → FetchEDGARFilings(dateRange) → []InsiderTrade
  → MergeAndDeduplicate(trades) → []InsiderTrade
  → Classify(trades) → []InsiderTrade
  → WriteInsiderTrades(trades)
  → PublishNATS(trades)  // publish to events.insider_trades

InsiderClusterWorkflow (cron: every 4 hours)
  → QueryRecentTrades(window: 7d) → []InsiderTrade
  → DetectClusters(trades) → []ClusterSignal
  → WriteInsiderTrades(clusters)
  → PublishNATS(clusters)  // publish to events.insider_trades
```

### Cluster Detection

Flag when 3+ insiders at the same company buy within 7 days (cluster buy signal). Also flag when insider buy/sell ratio for a sector shifts significantly.

### Urgency Classification

- **alert**: CEO/CFO purchase >$1M, cluster buy (3+ insiders), or insider selling >$10M
- **notable**: Any C-suite trade, director purchase >$100K
- **routine**: Small trades, option exercises, automated plans (10b5-1)

Filter out 10b5-1 pre-planned trades where possible (Finnhub has this flag).

### Config

```go
type Config struct {
    FinnhubAPIKey      string `env:"FINNHUB_API_KEY"`
    BrowserAPIURL      string `env:"BROWSER_API_URL" default:"http://localhost:8080"`
    FetchInterval      string `env:"INSIDER_FETCH_INTERVAL" default:"1h"`
    SECUserAgent       string `env:"SEC_USER_AGENT" default:"VoidAlpha admin@voidalpha.com"`
}
```

### Data Written

**Primary table: `insider_trades`** (UUID primary key)
- `insider_name`, `role`, `ticker`, `transaction_type`, `shares`, `price`, `total_value`, `is_10b5_1`, `filing_date`, `transaction_date`, `source_id` (UNIQUE)

After each write, publishes to NATS JetStream subject `events.insider_trades` with `{"id": "uuid", "table": "insider_trades"}` payload.

---

## 5.3 — finra-data Service

### What It Does
Tracks short interest data, dark pool (ATS) volume, and failure-to-deliver (FTD) data.

### Data Sources

#### FINRA Query API (structured, free)
- **Auth**: OAuth 2.0 with free public credentials
  - Token endpoint: `https://ews.fip.finra.org/fip/rest/ews/oauth2/access_token?grant_type=client_credentials`
- **Base URL**: `https://api.finra.org/data/group/{group}/name/{dataset}`
- **Rate Limits**: 1,200 sync req/min per IP, 5,000 record cap per sync request
- **Key Datasets**:
  - `equity/shortInterest` — bi-monthly short interest data (settlement date, ticker, shares short)
  - `otcMarket/weeklySummary` — ATS/dark pool weekly volume by ticker and venue
  - `equity/regShoDaily` — daily Reg SHO threshold list
- **Response**: JSON or pipe-delimited CSV
- **No browser scraping needed**

#### FINRA ATS (Dark Pool) Data
- Same Query API above: `otcMarket/weeklySummary` dataset
- **Data**: ATS name, ticker, total shares, total trades, week ending date
- **Schedule**: Weekly publication. Check every 12 hours.

#### SEC Failure-to-Deliver (FTD)
- **URL**: `https://www.sec.gov/data/foiadocsfailsdatahtm`
- **Method**: Download CSV files (direct HTTP, no browser needed)
- **Data**: Settlement date, CUSIP, ticker, quantity of FTDs, description, price
- **Schedule**: Published twice monthly. Check daily.
- **Notes**: High FTD counts can indicate short selling pressure or settlement issues.

#### Short Volume (daily, free CDN)
- **URL**: `https://cdn.finra.org/equity/regsho/daily/CNMSshvol{YYYYMMDD}.txt`
- **Method**: Direct HTTP download of pipe-delimited text files. No auth.
- **Columns**: `Date|Symbol|ShortVolume|ShortExemptVolume|TotalVolume|Market`
- **Venue files**: `CNMS`, `FNQC`, `FNRA`, `FNYX`, `FORF`
- **Schedule**: Posted by 6 PM ET same day
- **No browser needed**

### Temporal Workflows

```
ShortInterestWorkflow (cron: every 12 hours)
  → FetchFINRAToken() → OAuth token
  → FetchShortInterest(token) → []ShortInterestData  // FINRA Query API
  → DetectChanges(current, previous) → []Change
  → WriteShortInterest(changes)
  → PublishNATS(changes)  // publish to events.short_interest

DarkPoolWorkflow (cron: every 12 hours)
  → FetchFINRAToken() → OAuth token
  → FetchATSData(token) → []DarkPoolData  // FINRA Query API
  → CalculateRatios(data) → []DarkPoolAnalysis
  → WriteDarkPoolVolume(analysis)
  → PublishNATS(analysis)  // publish to events.dark_pool_volume

ShortVolumeWorkflow (cron: daily at 6 PM ET)
  → DownloadShortVolume(date) → []ShortVolumeData
  → CalculateRatios(data) → []ShortVolumeAnalysis
  → WriteShortInterest(analysis)
  → PublishNATS(analysis)  // publish to events.short_interest

FTDWorkflow (cron: daily)
  → DownloadFTD() → []FTDData
  → FilterSignificant(data, threshold) → []FTDData
  → WriteFTDData(data)
  → PublishNATS(data)  // publish to events.ftd_data
```

### Urgency Classification

- **alert**: Short interest >30% of float, or short interest increase >50% in one reporting period
- **notable**: Short interest >20% of float, dark pool volume >50% of total volume for a ticker
- **routine**: Regular updates

### Config

```go
type Config struct {
    BrowserAPIURL      string `env:"BROWSER_API_URL" default:"http://localhost:8080"`
    SECUserAgent       string `env:"SEC_USER_AGENT" default:"VoidAlpha admin@voidalpha.com"`
    ShortInterestInterval string `env:"FINRA_SHORT_INTERVAL" default:"12h"`
    DarkPoolInterval   string `env:"FINRA_DARKPOOL_INTERVAL" default:"12h"`
}
```

### Data Written

Writes to THREE dedicated tables (all UUID primary keys), then publishes to NATS JetStream.

**Table: `short_interest`**
- `ticker`, `shares_short`, `float_pct`, `days_to_cover`, `change_pct`, `settlement_date`

**Table: `dark_pool_volume`**
- `ticker`, `ats_volume`, `total_volume`, `dark_pct`, `top_ats` (JSONB), `week_ending`

**Table: `ftd_data`**
- `ticker`, `quantity`, `price`, `notional_value`, `settlement_date`

After each write, publishes to the corresponding NATS JetStream subject (`events.short_interest`, `events.dark_pool_volume`, or `events.ftd_data`) with `{"id": "uuid", "table": "<table_name>"}` payload.

---

## 5.4 — Deliverables

After this phase:
- [ ] `congress-trades` scraping Capitol Trades + Quiver Quant, detecting cluster signals
- [ ] `insider-trading` pulling SEC EDGAR Form 4 + Finnhub, detecting cluster buys
- [ ] `finra-data` tracking short interest, dark pool volume, FTDs, daily short volume
- [ ] All three services using stealthy-auto-browse for JS-heavy scraping targets
- [ ] Urgency classification for large/unusual trades and positions
- [ ] Cross-referencing: Congress trades linked to committee oversight, insiders linked to earnings dates
