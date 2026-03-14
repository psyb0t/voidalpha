# 04 — Macro & Fed

Economic indicators, GDP, CPI, unemployment, Fed speeches, FOMC minutes, rate decisions, Treasury yields, economic calendar.

## Services in This Phase

| Service | Task Queue | Schedule |
|---------|-----------|----------|
| `macro-data` | `macro-data-queue` | Every 1–6 hours (varies by indicator) |
| `fed-watcher` | `fed-watcher-queue` | Every 15–60 min |
| `econ-calendar` | `econ-calendar-queue` | Every 6 hours |

---

## 4.1 — macro-data Service

### What It Does
Fetches US economic indicators from FRED, BLS, and BEA. Tracks releases, detects significant changes, writes to `macro_indicators` table and publishes notifications to NATS JetStream.

### Data Sources

#### Primary: FRED API v2 (Federal Reserve Bank of St. Louis)
- **Base URL**: `https://api.stlouisfed.org/fred`
- **Auth**: `api_key` query param or `Authorization: Bearer` header (v2, launched Nov 2025)
- **Registration**: Free at `https://fredaccount.stlouisfed.org`
- **Key Endpoints**:
  - `GET /series/observations?series_id=GDP&api_key=X&file_type=json` — time series data (values are strings, missing = `"."`)
  - `GET /series?series_id=GDP&api_key=X&file_type=json` — series metadata
  - `GET /releases/dates?api_key=X&file_type=json&realtime_start=2026-03-01` — upcoming releases
- **Rate Limit**: No documented hard cap. Practically unlimited for reasonable usage.
- **Response**: JSON/XML/CSV. All numeric values returned as strings — parse accordingly.
- **Env Var**: `FRED_API_KEY`
- **Notes**: VIX also available here as series `VIXCLS` — no need to scrape CBOE for VIX data.

**Key FRED Series to Track**:

| Series ID | Name | Frequency |
|-----------|------|-----------|
| `GDP` | Gross Domestic Product | Quarterly |
| `CPIAUCSL` | Consumer Price Index (CPI) | Monthly |
| `UNRATE` | Unemployment Rate | Monthly |
| `FEDFUNDS` | Federal Funds Rate | Daily |
| `DFF` | Effective Federal Funds Rate | Daily |
| `T10Y2Y` | 10-Year minus 2-Year Treasury Spread | Daily |
| `T10YIE` | 10-Year Breakeven Inflation Rate | Daily |
| `PAYEMS` | Total Nonfarm Payrolls | Monthly |
| `RSAFS` | Retail Sales | Monthly |
| `HOUST` | Housing Starts | Monthly |
| `UMCSENT` | Consumer Sentiment (UMich) | Monthly |
| `DCOILWTICO` | WTI Crude Oil Price | Daily |
| `BAMLH0A0HYM2` | High Yield Spread (ICE BofA) | Daily |
| `DGS10` | 10-Year Treasury Rate | Daily |
| `DGS2` | 2-Year Treasury Rate | Daily |
| `M2SL` | M2 Money Supply | Monthly |
| `WALCL` | Fed Balance Sheet (Total Assets) | Weekly |
| `VIXCLS` | VIX (CBOE Volatility Index) | Daily |

#### Secondary: BLS API (Bureau of Labor Statistics)
- **Base URL**: `https://api.bls.gov/publicAPI/v2`
- **Auth**: `registrationkey` in POST body (optional for v2, required for higher limits)
- **Registration**: Free at `https://data.bls.gov/registrationEngine/`
- **Key Endpoint**:
  - `POST /timeseries/data/` — fetch one or more series
  - Body: `{"seriesid": ["CES0000000001"], "startyear": "2025", "endyear": "2026", "registrationkey": "X"}`
- **Rate Limits**:
  - v1 (no key): 25 queries/day, 25 series/query, 10 years/query
  - v2 (free key): 500 queries/day, 50 series/query, 20 years/query
- **Response**: JSON. Period codes like `M01`-`M12` for monthly data.
- **Env Var**: `BLS_API_KEY`

**Key BLS Series**:

| Series ID | Name |
|-----------|------|
| `CES0000000001` | Total Nonfarm Payrolls |
| `LNS14000000` | Unemployment Rate |
| `CUUR0000SA0` | CPI-U (All Urban Consumers) |
| `WPUFD49104` | PPI Final Demand |

#### Treasury.gov (two sources)

**FiscalData API**:
- **URL**: `https://api.fiscaldata.treasury.gov/services/api/fiscal_service`
- **Auth**: None (public, no API key)
- **Key Endpoints**:
  - `GET /v2/accounting/od/avg_interest_rates?sort=-record_date&page[size]=10` — average interest rates
  - `GET /v1/accounting/od/auctions_query` — auction data
  - `GET /v2/accounting/od/debt_to_penny?sort=-record_date&page[size]=1` — national debt
- **Response**: JSON with `data` array + pagination meta. Also supports `?format=csv`.

**Daily Yield Curve XML Feed**:
- **URL**: `https://home.treasury.gov/resource-center/data-chart-center/interest-rates/pages/xml?data=daily_treasury_yield_curve&field_tdr_date_value=2026`
- **Auth**: None
- **Data**: Daily yields for all maturities (`BC_1MONTH` through `BC_30YEAR`)
- **Format**: XML (OData EDM). Pagination via `page=N` (zero-indexed).
- **Also available**: `daily_treasury_bill_rates`, `daily_treasury_real_yield_curve`

- **Schedule**: Daily check

### Temporal Workflows

```
DailyIndicatorsWorkflow (cron: every 1 hour)
  → FetchFREDSeries(dailySeries) → map[string]Observation
  → DetectChanges(observations, previousValues) → []Change
  → WriteMacroIndicators(changes)
  → PublishNATS(changes)  // publish to events.macro_indicators

MonthlyReleasesWorkflow (cron: every 6 hours)
  → FetchFREDReleaseDates(range) → []Release
  → FetchBLSData(series, year) → []Observation
  → CompareToExpectations(actuals, estimates) → []Surprise
  → WriteMacroIndicators(surprises)
  → PublishNATS(surprises)  // publish to events.macro_indicators

TreasuryWorkflow (cron: every 12 hours)
  → FetchTreasuryRates() → TreasuryData
  → WriteMacroIndicators(data)
  → PublishNATS(data)  // publish to events.macro_indicators
```

### Change Detection

For each indicator, track the previous value. Generate events based on significance:
- **routine**: Regular update, within expectations
- **notable**: Monthly release with deviation from consensus >1 std dev
- **alert**: Major surprise (e.g., CPI spike, payrolls miss by >100k, recession indicators flip)

Yield curve inversion (`T10Y2Y < 0`) always generates an `alert`.

### Config

```go
type Config struct {
    FREDAPIKey         string `env:"FRED_API_KEY"`
    BLSAPIKey          string `env:"BLS_API_KEY"`
    DailyInterval      string `env:"MACRO_DAILY_INTERVAL" default:"1h"`
    MonthlyInterval    string `env:"MACRO_MONTHLY_INTERVAL" default:"6h"`
    TrackedFREDSeries  string `env:"MACRO_FRED_SERIES" default:""`  // override defaults
}
```

### Data Written

**Primary table: `macro_indicators`** (UUID primary key)
- `id` UUID PK
- `series_id` TEXT — e.g. `'CPIAUCSL'`, `'T10Y2Y'`, `'PAYEMS'`
- `value` DOUBLE PRECISION
- `previous_value` DOUBLE PRECISION
- `change_pct` DOUBLE PRECISION
- `yoy_pct` DOUBLE PRECISION
- `source` TEXT — e.g. `'fred'`, `'bls'`, `'treasury'`
- `observed_at` TIMESTAMPTZ

After each write, publishes to NATS JetStream subject `events.macro_indicators` with `{"id": "uuid", "table": "macro_indicators"}` payload.

---

## 4.2 — fed-watcher Service

### What It Does
Monitors Federal Reserve communications: speeches, FOMC minutes, rate decisions, dot plots, press conferences.

### Data Sources

#### Fed RSS Feeds (no API key)
- **Press Releases**: `https://www.federalreserve.gov/feeds/press_all.xml`
- **Speeches**: `https://www.federalreserve.gov/feeds/speeches.xml`
- **Testimony**: `https://www.federalreserve.gov/feeds/testimony.xml`
- **Board Actions**: `https://www.federalreserve.gov/feeds/boardvotes.xml`
- **Method**: Standard RSS parsing, check every 15 min

#### FOMC Calendar
- **URL**: `https://www.federalreserve.gov/monetarypolicy/fomccalendars.htm`
- **Method**: Scrape with stealthy-auto-browse (static page, but HTML table)
- **Data**: Meeting dates, minutes release dates, press conference dates
- **Schedule**: Daily check (calendar changes rarely)

#### Fed Speeches (full text)
- When RSS detects a new speech:
  1. Fetch the speech page URL (most are plain HTML, use direct HTTP + goquery)
  2. Use stealthy-auto-browse only if direct fetch fails
  3. Store the full text in the `fed_speeches` table

#### CME FedWatch (rate expectations)
- **URL**: `https://www.cmegroup.com/markets/interest-rates/cme-fedwatch-tool.html`
- **Method**: Scrape with stealthy-auto-browse (JS-heavy)
- **Data**: Probability of rate hike/cut at each upcoming meeting
- **Schedule**: Every 2 hours

### Temporal Workflows

```
FedRSSWorkflow (cron: every 15 min)
  → FetchFedRSS(feeds) → []FedItem
  → Deduplicate(items) → []FedItem
  → For each new item:
    → FetchFullText(url) → string
  → WriteFedSpeeches(items)
  → PublishNATS(items)  // publish to events.fed_speeches

FOMCCalendarWorkflow (cron: every 24 hours)
  → ScrapeFOMCCalendar() → []FOMCMeeting
  → WriteFOMCMeetings(meetings)
  → PublishNATS(meetings)  // publish to events.fomc_meetings

FedWatchWorkflow (cron: every 2 hours)
  → ScrapeFedWatch() → RateExpectations
  → DetectShift(current, previous) → bool
  → WriteRateExpectations(expectations)
  → PublishNATS(expectations)  // publish to events.rate_expectations, notable if >5% shift
```

### Config

```go
type Config struct {
    BrowserAPIURL      string `env:"BROWSER_API_URL" default:"http://localhost:8080"`
    RSSInterval        string `env:"FED_RSS_INTERVAL" default:"15m"`
    FedWatchInterval   string `env:"FED_WATCH_INTERVAL" default:"2h"`
}
```

### Data Written

**Writes to THREE dedicated tables** (all UUID primary keys), depending on event type:

**1. `fed_speeches`** — for Fed speeches
- `id` UUID PK
- `speaker` TEXT — e.g. `'Jerome Powell'`
- `speech_type` TEXT — e.g. `'speech'`, `'testimony'`, `'press_conference'`
- `title` TEXT
- `url` TEXT
- `full_text` TEXT
- `published_at` TIMESTAMPTZ

**2. `fomc_meetings`** — for FOMC calendar entries
- `id` UUID PK
- `meeting_start` DATE
- `meeting_end` DATE
- `minutes_release` DATE
- `press_conference` BOOLEAN

**3. `rate_expectations`** — for CME FedWatch rate probabilities
- `id` UUID PK
- `meeting_date` DATE
- `hold_pct` DOUBLE PRECISION
- `cut_25_pct` DOUBLE PRECISION
- `cut_50_pct` DOUBLE PRECISION
- `hike_25_pct` DOUBLE PRECISION
- `shift_from_prev` DOUBLE PRECISION

After each write, publishes to the corresponding NATS JetStream subject (`events.fed_speeches`, `events.fomc_meetings`, or `events.rate_expectations`) with `{"id": "uuid", "table": "<table_name>"}` payload.

---

## 4.3 — econ-calendar Service

### What It Does
Fetches upcoming economic data releases (CPI, NFP, GDP, FOMC, etc.) with dates, times, previous values, forecasts, and importance. Writes to `econ_calendar` table and publishes notifications to NATS JetStream. Swing traders check this to know what's coming and when.

### Data Sources

#### Primary: Finnhub Economic Calendar
- **Endpoint**: `GET /calendar/economic?from=2026-03-13&to=2026-03-20`
- **Returns**: `[{country, event, impact, prev, estimate, actual, unit, time}]`
- **Free Tier**: Included
- **Env Var**: `FINNHUB_API_KEY` (shared)
- **Notes**: Covers US + international. Filter to `country: "US"` for phase 1.

#### Secondary: FRED Release Calendar
- **Endpoint**: `GET /releases/dates?api_key=X&file_type=json&realtime_start=2026-03-13&include_release_dates_with_no_data=true`
- **Returns**: Release dates for all FRED data series
- **Env Var**: `FRED_API_KEY` (shared)
- **Notes**: Cross-reference with Finnhub to fill gaps.

#### Tertiary: Investing.com Economic Calendar (scrape)
- **URL**: `https://www.investing.com/economic-calendar/`
- **Method**: Direct HTTP + goquery (HTML tables, no JS rendering needed for the calendar data). Set a proper `User-Agent`.
- **Data**: Event name, date/time, impact (bull icons 1-3), previous, forecast, actual
- **Notes**: Fallback if Finnhub data is incomplete. Good for importance ratings.

### Temporal Workflows

```
EconCalendarWorkflow (cron: every 6 hours)
  → FetchFinnhubCalendar(from, to) → []EconEvent
  → FetchFREDReleases(from, to) → []EconEvent
  → MergeAndDeduplicate(events) → []EconEvent
  → WriteEconCalendar(events)
  → PublishNATS(events)  // publish to events.econ_calendar
```

### Config

```go
type Config struct {
    FinnhubAPIKey  string `env:"FINNHUB_API_KEY"`
    FREDAPIKey     string `env:"FRED_API_KEY"`
    FetchInterval  string `env:"ECON_CALENDAR_INTERVAL" default:"6h"`
    LookAheadDays  int    `env:"ECON_CALENDAR_LOOKAHEAD" default:"14"`
}
```

### Data Written

**Primary table: `econ_calendar`** (UUID primary key)
- `id` UUID PK
- `event_name` TEXT — e.g. `'CPI (YoY)'`, `'FOMC Rate Decision'`
- `country` TEXT — e.g. `'US'`
- `scheduled_at` TIMESTAMPTZ — date and time of the release
- `impact` TEXT — `'high'`, `'medium'`, `'low'`
- `previous` TEXT — previous release value
- `forecast` TEXT — consensus forecast
- `source_id` TEXT UNIQUE — dedup key, e.g. `'econ-cal-CPI-YoY-2026-03-17'`

After each write, publishes to NATS JetStream subject `events.econ_calendar` with `{"id": "uuid", "table": "econ_calendar"}` payload.

---

## 4.4 — Deliverables

After this phase:
- [ ] `macro-data` fetching 18+ FRED series, BLS data, Treasury rates
- [ ] Change detection generating events for significant indicator moves
- [ ] Yield curve inversion alerts
- [ ] `fed-watcher` monitoring Fed RSS feeds, FOMC calendar, FedWatch probabilities
- [ ] Rate expectation tracking with shift detection
- [ ] `econ-calendar` showing upcoming releases with dates, times, forecasts, impact levels
- [ ] All macro, Fed, and calendar events published to NATS JetStream, consumed by api-gateway, pushed via WebSocket
