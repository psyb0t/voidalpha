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

`source_id` format: `congress-{member_slug}-{ticker}-{trade_date}-{amount_range}`

Example: `congress-nancy-pelosi-NVDA-2026-03-01-1M-5M`

### Temporal Workflows

```
CongressTradesWorkflow (cron: every 2 hours)
  → ScrapeCapitolTradesActivity(lastScrapeTime) → []CongTrade
  → DeduplicateActivity(trades) → []CongTrade
  → EnrichActivity(trades) → []CongTrade  // add committee info, sector context
  → ClassifyUrgencyActivity(trades) → []CongTrade
  → WriteDataEventsActivity(trades)

QuiverBackfillWorkflow (cron: every 4 hours)
  → ScrapeQuiverActivity() → []CongTrade
  → CrossReferenceActivity(trades) → []CongTrade  // only keep ones not in Capitol Trades
  → WriteDataEventsActivity(trades)
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

Table: `data_events`
- `source_service`: `'congress-trades'`
- `category`: `'political'`
- Examples:
  - `title: "Rep. Nancy Pelosi bought $1M-$5M of NVDA"`, `body: {"member": "Nancy Pelosi", "party": "D", "chamber": "House", "ticker": "NVDA", "trade_type": "purchase", "amount_range": "$1,000,001 - $5,000,000", "trade_date": "2026-02-28", "filed_date": "2026-03-10", "committees": ["Financial Services"], "source": "capitoltrades"}`
  - `title: "Congress Cluster Alert: 5 members bought defense stocks"`, `body: {"tickers": ["LMT", "RTX", "NOC"], "members": [...], "window": "48h", "total_amount": "$5M+"}`

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
  → FetchFinnhubInsiderActivity(tickers) → []InsiderTrade
  → FetchEDGARFilingsActivity(dateRange) → []InsiderTrade
  → MergeAndDeduplicateActivity(trades) → []InsiderTrade
  → ClassifyActivity(trades) → []InsiderTrade
  → WriteDataEventsActivity(trades)

InsiderClusterWorkflow (cron: every 4 hours)
  → QueryRecentTradesActivity(window: 7d) → []InsiderTrade
  → DetectClustersActivity(trades) → []ClusterSignal
  → WriteDataEventsActivity(clusters)
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

Table: `data_events`
- `source_service`: `'insider-trading'`
- `category`: `'insider'`
- Examples:
  - `title: "AAPL CEO Tim Cook sold 50,000 shares at $185"`, `body: {"insider": "Tim Cook", "role": "CEO", "ticker": "AAPL", "transaction": "sale", "shares": 50000, "price": 185.0, "total_value": 9250000, "is_10b5_1": true, "filing_date": "2026-03-10"}`
  - `title: "Insider Cluster Buy: 4 PLTR insiders purchased this week"`, `body: {"ticker": "PLTR", "insiders": [{"name": "...", "shares": 10000}, ...], "window_days": 7, "total_shares": 85000, "avg_price": 22.50}`

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
  → FetchFINRATokenActivity() → OAuth token
  → FetchShortInterestActivity(token) → []ShortInterestData  // FINRA Query API
  → DetectChangesActivity(current, previous) → []Change
  → WriteDataEventsActivity(changes)

DarkPoolWorkflow (cron: every 12 hours)
  → FetchFINRATokenActivity() → OAuth token
  → FetchATSDataActivity(token) → []DarkPoolData  // FINRA Query API
  → CalculateRatiosActivity(data) → []DarkPoolAnalysis
  → WriteDataEventsActivity(analysis)

ShortVolumeWorkflow (cron: daily at 6 PM ET)
  → DownloadShortVolumeActivity(date) → []ShortVolumeData
  → CalculateRatiosActivity(data) → []ShortVolumeAnalysis
  → WriteDataEventsActivity(analysis)

FTDWorkflow (cron: daily)
  → DownloadFTDActivity() → []FTDData
  → FilterSignificantActivity(data, threshold) → []FTDData
  → WriteDataEventsActivity(data)
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

Table: `data_events`
- `source_service`: `'finra-data'`
- `category`: varies (see below)

Examples:
- `category: 'options'`, `title: "GME Short Interest: 45% of float"`, `body: {"ticker": "GME", "short_interest": 45000000, "float": 100000000, "short_pct_float": 45.0, "days_to_cover": 3.2, "settlement_date": "2026-03-01", "change_pct": 12.0}`
- `category: 'options'`, `title: "SPY Dark Pool Volume: 55% of total"`, `body: {"ticker": "SPY", "dark_pool_volume": 150000000, "total_volume": 273000000, "dark_pct": 55.0, "top_ats": [{"name": "Citadel", "volume": 45000000}]}`
- `category: 'options'`, `title: "TSLA FTD Spike: 2.5M shares"`, `body: {"ticker": "TSLA", "ftd_quantity": 2500000, "price": 250.0, "notional_value": 625000000, "settlement_date": "2026-03-05"}`

---

## 5.4 — Deliverables

After this phase:
- [ ] `congress-trades` scraping Capitol Trades + Quiver Quant, detecting cluster signals
- [ ] `insider-trading` pulling SEC EDGAR Form 4 + Finnhub, detecting cluster buys
- [ ] `finra-data` tracking short interest, dark pool volume, FTDs, daily short volume
- [ ] All three services using stealthy-auto-browse for JS-heavy scraping targets
- [ ] Urgency classification for large/unusual trades and positions
- [ ] Cross-referencing: Congress trades linked to committee oversight, insiders linked to earnings dates
