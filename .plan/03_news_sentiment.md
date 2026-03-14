# 03 — News & Sentiment

Financial news aggregation, YouTube analyst transcripts, fear & greed tracking, social sentiment, VIX signals.

## Services in This Phase

| Service | Task Queue | Schedule |
|---------|-----------|----------|
| `news-aggregator` | `news-aggregator-queue` | Every 5 min |
| `youtube-intel` | `youtube-intel-queue` | Every 30 min |
| `sentiment-tracker` | `sentiment-tracker-queue` | Every 15–60 min (varies by source) |

---

## 3.1 — news-aggregator Service

### What It Does
Aggregates financial news from multiple sources, deduplicates, writes to `news_articles` table, and publishes notifications to NATS JetStream.

### Data Sources

#### RSS Feeds (no API key needed)
Direct HTTP fetch, parse XML. No browser needed.

Not just financial news — geopolitical, tech, energy, health, regulatory, trade wars, natural disasters, etc. all move markets.

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
  - `GET /everything?language=en&sortBy=publishedAt` — broad search
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

Dedup uses `source_id UNIQUE` constraint on `news_articles`. ON CONFLICT (source_id) DO NOTHING on `news_articles`. Only publish to NATS if the `news_articles` insert succeeded (returned an id).

### Temporal Workflows

```
NewsFetchWorkflow (cron: every 5 min)
  → FetchRSS(feeds []FeedConfig) → []Article
  → FetchFinnhubNews(category) → []Article
  → FetchNewsAPI(query) → []Article  // only if within daily limit
  → MergeAndDeduplicate(allArticles) → []Article
  → WriteArticles(articles)
  → PublishNATS(articles)  // publish to events.news_articles
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

**Table: `news_articles`** (UUID primary key)
- `headline`: Article headline
- `source_name`: e.g. `"Reuters"`, `"Finnhub"`, `"NewsAPI"`
- `summary`: Article summary text
- `url`: Article URL
- `image_url`: Article image URL (nullable)
- `body_text`: Full article body text (nullable)
- `published_at`: Article publish timestamp
- `source_id`: Dedup key (UNIQUE, see above)

After each write, publishes to NATS JetStream subject `events.news_articles` with `{"id": "uuid", "table": "news_articles"}` payload.

---

## 3.2 — youtube-intel Service

### What It Does
Downloads transcripts from a configurable list of YouTube channels using yt-dlp. Stores raw transcripts in `youtube_transcripts` for reference and analysis.

### Target Channels (configurable via env/config)

Macro analysts, news analysts, quants, traders — anything that might contain market-moving analysis:

| Channel | Focus |
|---------|-------|
| Meet Kevin | Macro, Fed, real estate, market commentary |
| Arete Trading | Technical analysis, options flow |
| Bravos Research | Macro, geopolitical, institutional flow |
| Eurodollar University (Jeff Snider) | Macro, bonds, liquidity |
| The Maverick of Wall Street | Daily market recap, technicals |
| Game of Trades | Macro, charts, recession signals |
| Heresy Financial | Macro, contrarian takes |
| Wealthion | Macro interviews, fund managers |
| Patrick Boyle | Quant finance, market structure |
| Add more via config... | |

Channel list is fully configurable — add/remove via `YOUTUBE_CHANNELS` env var (comma-separated channel IDs/URLs).

### How It Works

1. yt-dlp checks each channel's RSS feed or uploads page for new videos
2. For each new video not already in the DB (dedup by video ID):
   - Download auto-generated or manual subtitles/transcript (no video download)
   - `yt-dlp --write-auto-sub --sub-lang en --skip-download --write-sub`
   - Extract transcript text (VTT/SRT → plain text)
3. Write transcript to `youtube_transcripts` table
4. Publish notification to NATS JetStream subject `events.youtube_transcripts`

### Deduplication

`video_id` UNIQUE on `youtube_transcripts` (e.g., `dQw4w9WgXcQ`). Only publish to NATS if the `youtube_transcripts` insert succeeded (returned an id).

### Temporal Workflows

Parent workflow launches child workflows in parallel — one per channel. Each child handles its own videos independently.

```
YouTubeScanWorkflow (cron: every 30 min)
  → ListChannels() → []ChannelConfig
  → For each channel: launch ChannelFetchWorkflow as child workflow (parallel)

ChannelFetchWorkflow (child, one per channel)
  → FetchNewVideos(channelID) → []VideoMeta
  → For each new video (not in DB):
    → DownloadTranscript(videoID) → string
    → WriteTranscript(transcript)
    → PublishNATS(transcript)  // publish to events.youtube_transcripts
```

**Retry Policy**: 3 attempts, 15s initial, backoff 2.0, max 120s. Transcript download can be slow.

### Config

```go
type Config struct {
    Channels      string `env:"YOUTUBE_CHANNELS" default:""`  // comma-separated channel IDs or URLs
    FetchInterval string `env:"YOUTUBE_FETCH_INTERVAL" default:"30m"`
    SubLang       string `env:"YOUTUBE_SUB_LANG" default:"en"`
}
```

### Data Written

**Table: `youtube_transcripts`** (UUID primary key)
- `channel`: Channel name (e.g., `"Meet Kevin"`)
- `channel_id`: YouTube channel ID (e.g., `"UCUvvj5lwue7PwE-yL7r671A"`)
- `video_id`: YouTube video ID (UNIQUE, e.g., `"abc123"`)
- `title`: Video title (e.g., `"THE FED JUST BROKE EVERYTHING | Meet Kevin"`)
- `transcript`: Full transcript text
- `duration_secs`: Video duration in seconds (e.g., `1847`)
- `published_at`: Video publish timestamp

After each write, publishes to NATS JetStream subject `events.youtube_transcripts` with `{"id": "uuid", "table": "youtube_transcripts"}` payload.

### Notes
- yt-dlp needs to be installed in the Docker image (`pip install yt-dlp` or binary)
- No YouTube API key needed — yt-dlp works without auth for public videos/transcripts
- Only downloads subtitles, never the actual video — minimal bandwidth/storage
- Some videos won't have auto-generated subs (rare for English content) — skip those

---

## 3.3 — sentiment-tracker Service

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
- Publishes to NATS JetStream subject `events.sentiment_readings` when VIX crosses thresholds

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
  → FetchFearGreed() → FearGreedData
  → WriteSentimentReading(data)
  → PublishNATS(data)  // publish to events.sentiment_readings

PutCallRatioWorkflow (cron: every 30 min, market hours)
  → ScrapeCBOE() → PutCallData  // uses stealthy-auto-browse
  → WriteSentimentReading(data)
  → PublishNATS(data)  // publish to events.sentiment_readings

VIXAnalysisWorkflow (cron: every 15 min)
  → ReadLatestVIX() → VIXSnapshot  // reads from market_snapshots
  → ClassifyVIX(snapshot) → VIXClassification
  → WriteSentimentReading(classification)  // only if threshold crossed
  → PublishNATS(classification)  // publish to events.sentiment_readings

AAIISentimentWorkflow (cron: every 6 hours)
  → ScrapeAAII() → AAIIData  // uses stealthy-auto-browse
  → WriteSentimentReading(data)
  → PublishNATS(data)  // publish to events.sentiment_readings

RedditSentimentWorkflow (cron: every 30 min)
  → FetchRedditPosts(subreddits) → []Post
  → AnalyzeSentiment(posts) → SentimentReport
  → WriteSentimentReading(report)
  → PublishNATS(report)  // publish to events.sentiment_readings
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

**Table: `sentiment_readings`** (UUID primary key)
- `indicator`: Indicator name (e.g., `"fear_greed"`, `"put_call_ratio"`, `"vix"`, `"aaii"`, `"reddit"`)
- `value`: Numeric value (e.g., `25`, `1.35`, `31.5`)
- `label`: Human-readable label (e.g., `"extreme_fear"`, `"elevated"`, `"elevated_fear"`)
- `components` JSONB: Breakdown data (e.g., `{"previous": 32, ...}` for Fear & Greed, `{"total_pcr": 1.35, "equity_pcr": 1.20, "index_pcr": 1.50}` for put/call)
- `captured_at`: Timestamp when the reading was captured

After each write, publishes to NATS JetStream subject `events.sentiment_readings` with `{"id": "uuid", "table": "sentiment_readings"}` payload.

---

## 3.4 — Deliverables

After this phase:
- [ ] `news-aggregator` pulling from 8+ RSS feeds, Finnhub News, and NewsAPI
- [ ] News deduplication working via `source_id`
- [ ] `youtube-intel` downloading transcripts from configurable channel list via yt-dlp
- [ ] YouTube transcripts stored in `youtube_transcripts` table
- [ ] `sentiment-tracker` fetching Fear & Greed, put/call ratio, VIX, AAII, Reddit sentiment
- [ ] Browser scraping working for JS-heavy sources (CBOE, AAII)
- [ ] Sentiment events published to NATS JetStream, consumed by api-gateway, pushed via WebSocket to frontend
