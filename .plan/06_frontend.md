# 06 — Frontend

Svelte dashboard compiled to static HTML + JS. Real-time updates via WebSocket. No SSR, no Node in production.

## 6.1 — Tech Setup

### Build Pipeline
- **Svelte 5** (runes mode) with **SvelteKit** in static adapter mode
- **Vite** for dev server + build
- **TailwindCSS** for styling
- Output: `build/` directory with `index.html` + JS/CSS assets
- Served by `api-gateway` Go service as static files

### Project Structure

```
frontend/
├── package.json
├── svelte.config.js          # static adapter
├── vite.config.js
├── tailwind.config.js
├── src/
│   ├── app.html              # shell HTML
│   ├── app.css               # Tailwind imports
│   ├── lib/
│   │   ├── stores/
│   │   │   ├── events.js     # event notifications store (WebSocket-fed)
│   │   │   ├── snapshots.js  # market snapshots store (WebSocket-fed)
│   │   │   ├── filters.js    # active filters (ticker, source_service, etc.)
│   │   │   └── ws.js         # WebSocket connection manager
│   │   ├── components/
│   │   │   ├── EventCard.svelte
│   │   │   ├── MarketTicker.svelte
│   │   │   ├── GammaChart.svelte
│   │   │   ├── FearGreedGauge.svelte
│   │   │   ├── FilterBar.svelte
│   │   │   ├── Sidebar.svelte
│   │   │   ├── Header.svelte
│   │   │   └── ...
│   │   └── utils/
│   │       ├── format.js     # number/date formatting
│   │       └── api.js        # REST API client
│   └── routes/
│       ├── +layout.svelte    # app shell: sidebar + header
│       ├── +page.svelte      # / → Dashboard
│       ├── market/
│       │   └── +page.svelte  # /market → Market data + snapshots
│       ├── news/
│       │   └── +page.svelte  # /news → News feed
│       ├── options/
│       │   └── +page.svelte  # /options → Options flow + gamma
│       ├── macro/
│       │   └── +page.svelte  # /macro → Macro + Fed data
│       ├── insider/
│       │   └── +page.svelte  # /insider → Insider + Congress trades
│       └── sentiment/
│           └── +page.svelte  # /sentiment → Fear & Greed + social
```

### Build & Deploy

```bash
# Dev
cd frontend && npm install && npm run dev

# Build for production
npm run build
# Output: frontend/build/

# api-gateway serves frontend/build/ as static files at /
```

The `Dockerfile` builds the frontend as a build stage, then copies `build/` into the Go binary's container.

---

## 6.2 — WebSocket Integration

### Connection Manager (`ws.js`)

```
Connect to ws://host:3001/ws
On open: send subscription preferences (optional)
On message: parse JSON, route to appropriate store
On close: auto-reconnect with exponential backoff (1s, 2s, 4s, 8s, max 30s)
```

### Message Format (from api-gateway)

The api-gateway subscribes to NATS JetStream subjects and forwards messages to the frontend over WebSocket. The frontend has no knowledge of NATS — it only sees WebSocket JSON messages.

**Event notifications** (from NATS subject `events.{table_name}`):
```json
{
  "subject": "events.congress_trades",
  "id": "a1b2c3d4-...",
  "table": "congress_trades"
}
```

**Snapshot updates** (from NATS subject `snapshots.{ticker}`):
```json
{
  "subject": "snapshots.SPY",
  "id": "e5f6a7b8-...",
  "ticker": "SPY",
  "price": 520.50,
  "change_pct": -1.2
}
```

### Store Updates

WebSocket messages update Svelte stores. Components reactively re-render.

- **Event notifications** (`events.*` messages) → prepend to `$events` store (capped at 500 items in memory). Each notification is a pointer containing only `table` and `id`. The frontend fetches the actual displayable content (title, body, etc.) from the REST API (e.g., `GET /api/{table}/{id}`).
- **Snapshot updates** (`snapshots.*` messages) → upsert in `$snapshots` store (keyed by ticker). Snapshot data is inline in the message (price, change_pct, etc.) — no additional fetch needed.

---

## 6.3 — Pages

### Dashboard (`/`)

The main overview page. Shows everything at a glance.

**Layout**:
```
┌─────────────────────────────────────────────────┐
│ HEADER: VoidAlpha | Market Status | Clock        │
├────────┬────────────────────────────────────────┤
│        │ MARKET TICKER BAR (scrolling)          │
│        ├────────────────────────────────────────┤
│        │ ┌──────────┐ ┌──────────┐ ┌──────────┐│
│ SIDE   │ │ Fear &   │ │ VIX      │ │ Put/Call ││
│ BAR    │ │ Greed    │ │ Current  │ │ Ratio    ││
│        │ └──────────┘ └──────────┘ └──────────┘│
│ (nav)  ├────────────────────────────────────────┤
│        │ LIVE EVENT FEED                        │
│        │ [filter: source | ticker]               │
│        │                                        │
│        │ ● news-aggregator: Fed raises rates...   │
│        │ ● insider-tracker: NVDA insider buy...  │
│        │ ● congress-watcher: Rep. X bought AAPL  │
│        │ ● macro-collector: CPI January: 3.1%... │
│        │ ● market-data: GME short interest...     │
│        │ ...                                     │
└────────┴────────────────────────────────────────┘
```

**Components**:
- `MarketTicker` — horizontal scrolling bar with major indices, commodities, crypto (from `$snapshots`)
- `FearGreedGauge` — circular gauge showing CNN F&G score
- `EventCard` — individual event notification showing source, timestamp, and resolved detail (fetched from REST API via table/id)
- `FilterBar` — filter events by source service, ticker
- Live feed receives NATS-forwarded event notifications via WebSocket, fetches detail from REST endpoints, auto-scrolls with new events sliding in from top with subtle animation

### Market (`/market`)

**Data**: Market snapshots, price charts, movers

**Sections**:
- **Indices**: S&P 500, Nasdaq, Dow, Russell 2000 — card per index with price, change, mini sparkline
- **Commodities**: Gold, Silver, Oil, Nat Gas — same format
- **Crypto**: BTC, ETH
- **Watchlist**: User's tracked tickers (from config) with sortable table
- **Top Movers**: Biggest gainers/losers from snapshot data

**API calls**:
- `GET /api/snapshots?ticker=^GSPC,^IXIC,...` — latest snapshots
- WebSocket for live updates

### News (`/news`)

**Data**: News events from `news-aggregator`

**Layout**: Scrollable feed of news cards with:
- Source badge (Reuters, CNBC, Bloomberg, etc.)
- Headline + content preview
- Related tickers (clickable → filter)
- Timestamp

**Filters**: Source, ticker, date range

**API call**: `GET /api/news_articles?limit=50` (fetched from the dedicated table directly, or resolved from event notifications via table/id)

### Options (`/options`)

**Data**: Options flow, gamma exposure, unusual activity

**Sections**:
- **Gamma Exposure Chart**: Bar chart of GEX by strike for selected ticker
  - X-axis: strike prices
  - Y-axis: gamma exposure ($)
  - Current spot price marked
  - Max pain level marked
- **Options Flow Table**: Recent options activity sorted by volume/OI
- **Unusual Activity Feed**: High volume/OI options flagged by the system

**API calls**:
- `GET /api/gamma?ticker=SPY` — latest gamma exposure
- `GET /api/options?ticker=SPY` — options flow data
- `GET /api/events?source_service=options-collector` — unusual activity notifications (detail resolved via table/id)

### Macro (`/macro`)

**Data**: Economic indicators, Fed speeches, FOMC, rate expectations

**Sections**:
- **Key Indicators Dashboard**: Grid of cards for GDP, CPI, Unemployment, Fed Funds Rate, 10Y yield, etc.
  - Current value, previous value, change
  - Trend arrow (up/down)
  - Last updated timestamp
- **Yield Curve**: Simple line showing 2Y, 5Y, 10Y, 30Y rates. Highlight inversion.
- **Fed Activity Feed**: Recent Fed speeches, FOMC minutes, rate decisions
  - Hawkish/dovish score bar per speech
- **FedWatch**: Rate probability chart for next 3 meetings
- **Economic Calendar**: Upcoming releases with dates and expectations

**API call**: `GET /api/events?source_service=macro-collector&limit=50` (detail resolved via table/id)

### Insider (`/insider`)

**Data**: Insider trading + Congress trades

**Sections**:
- **Congress Trades Feed**: Cards with member photo placeholder, name, party, ticker, amount, trade type
  - Committee relevance indicator
  - Cluster detection alerts
- **Corporate Insider Feed**: Recent Form 4 filings
  - Filter by buy/sell, role (CEO, CFO, Director, 10% Owner)
  - Cluster buy signals highlighted
- **FINRA Data Panel**: Short interest leaders, dark pool volume, FTD spikes
  - Sortable table by short % of float

**Filters**: Trade type (buy/sell), role, amount range, party (Congress)

**API call**: `GET /api/events?source_service=insider-tracker,congress-watcher&limit=50` (detail resolved via table/id)

### Sentiment (`/sentiment`)

**Data**: Fear & Greed, AAII, put/call, VIX, Reddit

**Sections**:
- **Fear & Greed Gauge**: Large circular gauge (0-100) with label
  - Component breakdown: momentum, breadth, put/call, VIX, junk bonds, safe haven
- **VIX Panel**: Current VIX with classification (low/normal/elevated/extreme)
- **AAII Survey**: Bullish/Bearish/Neutral bar chart
- **Put/Call Ratio**: Current value with historical context
- **Reddit Buzz**: Top mentioned tickers with mention counts, sentiment score

**API call**: `GET /api/events?source_service=sentiment-collector&limit=50` (detail resolved via table/id)

---

## 6.4 — Timezone Handling

All backend timestamps are UTC. The frontend handles all timezone conversion:

- **Auto-detection**: On first load, detect timezone from `Intl.DateTimeFormat().resolvedOptions().timeZone`
- **User override**: Settings panel lets user pick a timezone manually (stored in `localStorage`)
- **Display**: All timestamps rendered through a shared `formatTime()` util that applies the active timezone
- **Econ calendar**: Event times (e.g., "CPI release 08:30 ET") come with `timezone` field in the body JSONB — frontend converts to user's timezone

---

## 6.5 — Responsive Design

- Desktop-first (primary use case: multi-monitor trading desk)
- Tablet: sidebar collapses to icons
- Mobile: sidebar hidden behind hamburger menu, single column layout
- Dark theme by default (trading UI convention)
- Color scheme: dark background (#0d1117), green for positive (#22c55e), red for negative (#ef4444), blue for neutral (#3b82f6), amber for alerts (#f59e0b)

---

## 6.6 — Charting

Use **Chart.js** or **Lightweight Charts** (TradingView open-source) for:
- Gamma exposure bar charts
- Yield curve line charts
- Fear & Greed gauge (custom SVG or canvas)
- Mini sparklines in market ticker cards

Preference: **Lightweight Charts** by TradingView for financial data (candlesticks, time series). **Chart.js** for simple bar/line/gauge charts.

---

## 6.7 — Deliverables

After this phase:
- [ ] Svelte app builds to static files, served by api-gateway
- [ ] WebSocket connection with auto-reconnect, feeding Svelte stores
- [ ] Dashboard page with live event feed, market ticker, sentiment gauges
- [ ] Market page with indices, commodities, crypto, watchlist
- [ ] News page with filterable news feed
- [ ] Options page with gamma exposure chart and options flow table
- [ ] Macro page with economic indicators, yield curve, Fed activity, FedWatch
- [ ] Insider page with Congress trades, corporate insider filings, FINRA data
- [ ] Sentiment page with Fear & Greed, VIX, AAII, put/call, Reddit
- [ ] Dark theme, responsive layout, TradingView charts
- [ ] Dockerfile multi-stage build: frontend build → Go binary → final image
