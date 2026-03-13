# 03 — News & Sentiment

Financial news aggregation, fear & greed tracking, social sentiment, VIX signals.

## Services in This Phase

| Service | Task Queue | Schedule |
|---------|-----------|----------|
| `news-aggregator` | `news-aggregator-queue` | Every 5 min |
| `sentiment-tracker` | `sentiment-tracker-queue` | Every 15–60 min (varies by source) |

---

## 3.1 — news-aggregator Service

### What It Does
Aggregates financial news from multiple sources, deduplicates, and writes to `data_events` with `category: 'news'`.

### Data Sources

#### RSS Feeds (no API key needed)
Direct HTTP fetch, parse XML. No browser needed.

Not just financial news — geopolitical, tech, energy, health, regulatory, trade wars, natural disasters, etc. all move markets. The categorizer (Claude Haiku) figures out what's market-relevant and tags it.

**Financial**:

| Feed | URL | Focus |
|------|-----|-------|
| Reuters Business | `https://www.reutersagency.com/feed/` | General financial news |
| CNBC Top News | `https://search.cnbc.com/rs/search/combinedcms/view.xml?partnerId=wrss01&id=100003114` | Market headlines |
| MarketWatch | `https://feeds.marketwatch.com/marketwatch/topstories/` | Market news |
| Bloomberg (via RSS) | `https://feeds.bloomberg.com/markets/news.rss` | Markets |
| Yahoo Finance | `https://finance.yahoo.com/news/rssindex` | General |
| Seeking Alpha | `https://seekingalpha.com/market_currents.xml` | Market currents |
| Federal Reserve | `https://www.federalreserve.gov/feeds/press_all.xml` | Fed press releases |
| SEC EDGAR | `https://www.sec.gov/cgi-bin/browse-edgar?action=getcurrent&type=8-K&dateb=&owner=include&count=40&search_text=&action=getcurrent&output=atom` | Material events |

**General / Geopolitical / World** (market-moving stuff comes from everywhere):

| Feed | URL | Focus |
|------|-----|-------|
| Reuters World | `https://www.reutersagency.com/feed/?taxonomy=world` | Geopolitics, conflicts, trade |
| AP News | `https://rsshub.app/apnews/topics/apf-topnews` | Breaking world news |
| BBC World | `http://feeds.bbci.co.uk/news/world/rss.xml` | Global events |
| Al Jazeera | `https://www.aljazeera.com/xml/rss/all.xml` | Middle East, energy regions |
| NPR News | `https://feeds.npr.org/1001/rss.xml` | US domestic policy |
| The Hill | `https://thehill.com/feed/` | US politics, regulation |
| Politico | `https://www.politico.com/rss/politicopicks.xml` | US policy, trade |
| TechCrunch | `https://techcrunch.com/feed/` | Tech industry, startups, AI |
| Ars Technica | `https://feeds.arstechnica.com/arstechnica/index` | Tech, science |
| Wired | `https://www.wired.com/feed/rss` | Tech, cybersecurity |

#### Finnhub News API
- **Endpoint**: `GET /news?category=general` and `GET /company-news?symbol=AAPL&from=2026-03-01&to=2026-03-12`
- **Returns**: `[{category, datetime, headline, id, image, related, source, summary, url}]`
- **Free Tier**: Included, 60 calls/min
- **Env Var**: `FINNHUB_API_KEY` (shared)

#### NewsAPI
- **Base URL**: `https://newsapi.org/v2`
- **Auth**: `X-Api-Key` header or `apiKey` query param
- **Key Endpoints**:
  - `GET /everything?language=en&sortBy=publishedAt` — broad search (categorizer decides relevance)
  - `GET /top-headlines?country=us` — top US headlines across all categories
  - `GET /top-headlines?category=business&country=us` — business headlines
- **Free Tier**: 100 requests/day, articles delayed by 24h on free plan. Developer plan: 1000 requests/day.
- **Env Var**: `NEWSAPI_KEY`
- **Notes**: Good for broad coverage. Free tier has 24h delay — fine for background enrichment.

### Deduplication Strategy

Each article gets a `source_id` for dedup:
- RSS: `rss-{feed_name}-{guid or link hash}`
- Finnhub: `finnhub-news-{id}`
- NewsAPI: `newsapi-{url hash}`

Before writing, check `UNIQUE(source_service, source_id)` constraint. ON CONFLICT DO NOTHING.

### No Inline AI Processing

This service just fetches and writes raw events. No summarization or classification here. The `categorizer` service handles all AI processing (summarization, urgency, tagging) after events land in the DB.

### Temporal Workflows

```
NewsFetchWorkflow (cron: every 5 min)
  → FetchRSSActivity(feeds []FeedConfig) → []Article
  → FetchFinnhubNewsActivity(category) → []Article
  → FetchNewsAPIActivity(query) → []Article  // only if within daily limit
  → MergeAndDeduplicateActivity(allArticles) → []Article
  → WriteDataEventsActivity(articles)
```

**Retry Policy**: 3 attempts, 10s initial, backoff 2.0, max 60s. RSS feeds that fail are logged but don't block others.

### Config

```go
type Config struct {
    FinnhubAPIKey     string `env:"FINNHUB_API_KEY"`
    NewsAPIKey        string `env:"NEWSAPI_KEY"`
    FetchInterval     string `env:"NEWS_FETCH_INTERVAL" default:"5m"`
    MaxArticlesPerRun int    `env:"NEWS_MAX_ARTICLES" default:"100"`
}
```

### Data Written

Table: `data_events`
- `source_service`: `'news-aggregator'`
- `category`: `'news'`
- `title`: Article headline
- `body` JSONB: `{"summary": "...", "source_name": "Reuters", "image_url": "...", "related_tickers": ["AAPL"]}`
- `source_url`: Article URL
- `published_at`: Article publish timestamp
- `source_id`: Dedup key (see above)

---

## 3.2 — sentiment-tracker Service

### What It Does
Tracks market sentiment indicators: Fear & Greed Index, put/call ratio, VIX analysis, AAII sentiment, and Reddit sentiment.

### Data Sources

#### CNN Fear & Greed Index
- **URL**: `https://production.dataviz.cnn.io/index/fearandgreed/graphdata/{YYYY-MM-DD}`
- **Auth**: None, but **may block bots** (HTTP 418 "I'm a teapot" for non-browser User-Agents)
- **Returns**: JSON with `fear_and_greed_historical.data` array of `{x: unix_ms, y: 0-100}` pairs. Component scores for 7 indicators.
- **Method**: Try direct HTTP GET with realistic `User-Agent` header first. If blocked (418), fall back to stealthy-auto-browse on `https://www.cnn.com/markets/fear-and-greed`.
- **Components** (can be reconstructed from free data if CNN blocks entirely):
  1. S&P 500 vs 125-day MA (price momentum)
  2. NYSE 52-week highs vs lows (stock price strength)
  3. McClellan Volume Summation Index (breadth)
  4. CBOE put/call ratio 5-day avg
  5. VIX vs 50-day MA (volatility)
  6. Junk vs investment grade bond spread
  7. Stocks vs Treasuries 20-day return delta
- **Schedule**: Every 30 min

#### CBOE Put/Call Ratio
- **Historical CSV (no auth, direct download)**:
  - Total P/C: `https://cdn.cboe.com/resources/options/volume_and_call_put_ratios/totalpc.csv`
  - Index P/C: `https://cdn.cboe.com/resources/options/volume_and_call_put_ratios/indexpc.csv`
  - Equity P/C: `https://cdn.cboe.com/resources/options/volume_and_call_put_ratios/equitypc.csv`
  - Columns: `DATE, CALLS, PUTS, TOTAL, P/C Ratio` (data from 2006)
- **Current/live P/C**: Scrape `https://www.cboe.com/us/options/market_statistics/daily/` with stealthy-auto-browse (JS-rendered page)
- **Data**: Total P/C ratio, equity P/C ratio, index P/C ratio
- **Schedule**: Download CSVs daily EOD, scrape live page every 30 min during market hours

#### VIX Analysis
- Available from FRED as series `VIXCLS`, from `market-data` as `^VIX` in `market_snapshots`, and CBOE VIX history CSV at `https://cdn.cboe.com/api/global/us_indices/daily_prices/VIX_History.csv` (free, daily, back to 1990)
- Sentiment service reads latest VIX from DB, classifies:
  - VIX < 15: Low fear (complacency)
  - VIX 15–25: Normal
  - VIX 25–35: Elevated fear
  - VIX > 35: Extreme fear / panic
- Writes a `data_event` with `category: 'sentiment'` when VIX crosses thresholds

#### AAII Sentiment Survey
- **URL**: `https://www.aaii.com/sentimentsurvey/sent_results`
- **Auth**: Requires free AAII membership login (session cookies). Returns 403 without auth.
- **Method**: Login via stealthy-auto-browse, download Excel spreadsheet with results
- **Data**: Bullish %, Bearish %, Neutral %, 8-week moving averages, history back to 1987
- **Schedule**: Every Thursday (weekly release) + once daily to check
- **Notes**: Published every Thursday morning. No formal API — spreadsheet-based only.

#### Reddit Sentiment (optional, low priority)
- **API**: `https://oauth.reddit.com`
- **Auth**: OAuth2 — requires app registration at `https://www.reddit.com/prefs/apps`
- **Key Subreddits**: `r/wallstreetbets`, `r/stocks`, `r/investing`, `r/options`
- **Endpoints**:
  - `GET /r/{subreddit}/hot?limit=25` — hot posts
  - `GET /r/{subreddit}/comments/{article}` — post comments
- **Rate Limit**: 60 requests/min with OAuth
- **Env Vars**: `REDDIT_CLIENT_ID`, `REDDIT_CLIENT_SECRET`
- **Analysis**: Count ticker mentions, simple positive/negative keyword scoring
- **Schedule**: Every 30 min
- **Notes**: Low signal-to-noise. Useful for detecting retail momentum plays (meme stocks).

### Temporal Workflows

```
FearGreedWorkflow (cron: every 30 min)
  → FetchFearGreedActivity() → FearGreedData
  → WriteDataEventActivity(data)  // category: 'sentiment'

PutCallRatioWorkflow (cron: every 30 min, market hours)
  → ScrapeCBOEActivity() → PutCallData  // uses stealthy-auto-browse
  → WriteDataEventActivity(data)

VIXAnalysisWorkflow (cron: every 15 min)
  → ReadLatestVIXActivity() → VIXSnapshot  // reads from market_snapshots
  → ClassifyVIXActivity(snapshot) → VIXClassification
  → WriteDataEventActivity(classification)  // only if threshold crossed

AAIISentimentWorkflow (cron: every 6 hours)
  → ScrapeAAIIActivity() → AAIIData  // uses stealthy-auto-browse
  → WriteDataEventActivity(data)

RedditSentimentWorkflow (cron: every 30 min)
  → FetchRedditPostsActivity(subreddits) → []Post
  → AnalyzeSentimentActivity(posts) → SentimentReport
  → WriteDataEventActivity(report)
```

### Browser Scraping Pattern

For sources needing stealthy-auto-browse:

```
1. POST /action {action: "goto", url: "https://target.com/page"}
2. Wait for page load
3. POST /action {action: "get_text"} → extract data from DOM
4. Parse extracted text
5. (Container auto-stops or reuse for next scrape)
```

Spin up a dedicated browser container per scrape session. Don't share between concurrent scrapes.

### Config

```go
type Config struct {
    RedditClientID     string `env:"REDDIT_CLIENT_ID"`
    RedditClientSecret string `env:"REDDIT_CLIENT_SECRET"`
    BrowserAPIURL      string `env:"BROWSER_API_URL" default:"http://localhost:8080"`
    FearGreedInterval  string `env:"SENTIMENT_FEAR_GREED_INTERVAL" default:"30m"`
    RedditInterval     string `env:"SENTIMENT_REDDIT_INTERVAL" default:"30m"`
}
```

### Data Written

Table: `data_events`
- `source_service`: `'sentiment-tracker'`
- `category`: `'sentiment'`
- Examples:
  - `title: "Fear & Greed Index: 25 (Extreme Fear)"`, `body: {"score": 25, "label": "extreme_fear", "components": {...}, "previous": 32}`
  - `title: "Put/Call Ratio: 1.35 (Elevated)"`, `body: {"total_pcr": 1.35, "equity_pcr": 1.20, "index_pcr": 1.50}`
  - `title: "VIX Alert: Crossed above 30"`, `body: {"vix": 31.5, "classification": "elevated_fear", "threshold_crossed": 30}`
  - `title: "AAII Sentiment: 45% Bearish"`, `body: {"bullish": 25.5, "bearish": 45.0, "neutral": 29.5}`
  - `title: "Reddit: GME trending on r/wallstreetbets"`, `body: {"subreddit": "wallstreetbets", "top_tickers": [{"ticker": "GME", "mentions": 342}], "overall_sentiment": "bullish"}`

---

## 3.3 — Deliverables

After this phase:
- [ ] `news-aggregator` pulling from 8+ RSS feeds, Finnhub News, and NewsAPI
- [ ] News deduplication working via `source_id`
- [ ] Basic keyword urgency classification
- [ ] `sentiment-tracker` fetching Fear & Greed, put/call ratio, VIX, AAII, Reddit sentiment
- [ ] Browser scraping working for JS-heavy sources (CBOE, AAII)
- [ ] Sentiment events flowing to frontend via WebSocket
