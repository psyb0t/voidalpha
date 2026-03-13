# 07 — Frontend

Svelte dashboard compiled to static HTML + JS. Real-time updates via WebSocket. No SSR, no Node in production.

## 7.1 — Tech Setup

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
│   │   │   ├── events.js     # data_events store (WebSocket-fed)
│   │   │   ├── snapshots.js  # market_snapshots store
│   │   │   ├── filters.js    # active filters (ticker, category, urgency)
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

## 7.2 — WebSocket Integration

### Connection Manager (`ws.js`)

```
Connect to ws://host:3001/ws
On open: send subscription preferences (optional)
On message: parse JSON, route to appropriate store
On close: auto-reconnect with exponential backoff (1s, 2s, 4s, 8s, max 30s)
```

### Message Format (from api-gateway)

```json
{
  "channel": "data_events",
  "payload": {
    "id": 123,
    "source_service": "news-aggregator",
    "category": "news",
    "urgency": "notable",
    "title": "Fed raises rates by 25bp"
  }
}
```

```json
{
  "channel": "market_snapshots",
  "payload": {
    "id": 456,
    "ticker": "SPY",
    "price": 520.50,
    "change_pct": -1.2
  }
}
```

### Store Updates

WebSocket messages update Svelte stores. Components reactively re-render.

- `data_events` messages → prepend to `$events` store (capped at 500 items in memory)
- `market_snapshots` messages → upsert in `$snapshots` store (keyed by ticker)

---

## 7.3 — Pages

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
│        │ [filter: category | urgency | ticker]  │
│        │                                        │
│        │ ● ALERT: Fed raises rates by 25bp      │
│        │ ● NOTABLE: NVDA insider buy $2M        │
│        │ ○ Congress: Rep. X bought AAPL          │
│        │ ○ CPI January: 3.1% YoY                │
│        │ ○ GME short interest: 45% of float      │
│        │ ...                                     │
└────────┴────────────────────────────────────────┘
```

**Components**:
- `MarketTicker` — horizontal scrolling bar with major indices, commodities, crypto (from `$snapshots`)
- `FearGreedGauge` — circular gauge showing CNN F&G score
- `EventCard` — individual event with urgency indicator, category badge, timestamp, tags
- `FilterBar` — filter events by category, urgency, ticker
- Live feed auto-scrolls, new events slide in from top with subtle animation

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
- Headline + summary
- Related tickers (clickable → filter)
- Timestamp
- Urgency indicator

**Filters**: Source, urgency, ticker, date range

**API call**: `GET /api/events?category=news&limit=50`

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
- `GET /api/events?category=options` — unusual activity events

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

**API call**: `GET /api/events?category=macro&limit=50`

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

**API call**: `GET /api/events?category=insider,political&limit=50`

### Sentiment (`/sentiment`)

**Data**: Fear & Greed, AAII, put/call, VIX, Reddit

**Sections**:
- **Fear & Greed Gauge**: Large circular gauge (0-100) with label
  - Component breakdown: momentum, breadth, put/call, VIX, junk bonds, safe haven
- **VIX Panel**: Current VIX with classification (low/normal/elevated/extreme)
- **AAII Survey**: Bullish/Bearish/Neutral bar chart
- **Put/Call Ratio**: Current value with historical context
- **Reddit Buzz**: Top mentioned tickers with mention counts, sentiment score

**API call**: `GET /api/events?category=sentiment&limit=50`

---

## 7.4 — Responsive Design

- Desktop-first (primary use case: multi-monitor trading desk)
- Tablet: sidebar collapses to icons
- Mobile: sidebar hidden behind hamburger menu, single column layout
- Dark theme by default (trading UI convention)
- Color scheme: dark background (#0d1117), green for positive (#22c55e), red for negative (#ef4444), blue for neutral (#3b82f6), amber for alerts (#f59e0b)

---

## 7.5 — Charting

Use **Chart.js** or **Lightweight Charts** (TradingView open-source) for:
- Gamma exposure bar charts
- Yield curve line charts
- Fear & Greed gauge (custom SVG or canvas)
- Mini sparklines in market ticker cards

Preference: **Lightweight Charts** by TradingView for financial data (candlesticks, time series). **Chart.js** for simple bar/line/gauge charts.

---

## 7.6 — Deliverables

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
