# 100xSystems Engineering Discovery Platform — Status & Roadmap

> **Date:** July 23, 2026
> **Author:** AI-assisted planning + research
> **Constraint:** Zero cost, serverless only, no VPS, Next.js + Vercel Hobby + Turso + Clerk

---

## Part 1: What's Been Built

### 1.1 Feed Registry — 51 Engineering Blogs

A curated directory of engineering blog RSS feeds in `src/feed/feed.constants.ts`. Each entry stores only metadata (name, RSS URL, tags) — never article content.

**Coverage by category:**

| Category | Feeds | Examples |
|----------|-------|---------|
| **Large-scale infra** | 11 | Netflix, Stripe, Cloudflare, Discord, Uber, Meta, Slack, LinkedIn, Dropbox, Airbnb, Spotify |
| **Cloud/Infrastructure** | 7 | AWS Architecture, HashiCorp, Docker, DigitalOcean, Red Hat, NGINX, Cloudflare Workers |
| **Databases & Data** | 7 | CockroachDB, ClickHouse, Redis, PostgreSQL, MongoDB, Elastic, Confluent, Databricks |
| **Observability** | 3 | Grafana Labs, Datadog, Sentry |
| **Networking/Security** | 4 | Tailscale, Traefik, Istio, Cloudflare |
| **Frontend/DevTools** | 7 | Figma, Svelte, Vercel, GitHub, Deno, Bun, Sourcegraph, Stack Overflow, JetBrains |
| **Systems/Performance** | 5 | Rust, Julia Evans, Dan Luu, NASA Software, Pinecone |
| **AI/ML** | 2 | Apple ML Research, Databricks |

**Tag taxonomy:** 40+ tags auto-extracted via `ALL_TAGS` — covering `distributed-systems`, `databases`, `networking`, `security`, `frontend`, `backend`, `observability`, `ai`, `performance`, `infrastructure`, `cloud-native`, and more.

### 1.2 API Route — `GET /api/feed`

A serverless Next.js API route that:

- Accepts `?feeds=id1,id2&limit=N` query parameters
- Fetches RSS/Atom feeds in parallel using `Promise.allSettled` (partial failure tolerance)
- Parses XML via `rss-parser` library
- Returns JSON with article metadata only (title, URL, author, summary, publish date, tags)
- **Never stores or caches article content** (ethical by design)
- Sets `Cache-Control: public, s-maxage=300, stale-while-revalidate=3600` for edge caching
- Returns errors per-feed without failing the whole request

**Confirmed working** with real RSS data (tested with Cloudflare, Airbnb, and others).

### 1.3 Feed UI — `/feed`

| Component | File | Purpose |
|-----------|------|---------|
| `FeedPage.tsx` | Main orchestrator | State management, preferences, keyboard nav, reading history |
| `FeedHeader.tsx` | Controls | Feed source selector, sort toggle, tag filters, refresh |
| `ArticleCard.tsx` | Display | Article with bookmark, share, open actions + read/focus states |
| `FeedSourceSelector.tsx` | Blog picker | Dropdown with search, select all/clear, feed metadata |
| `FeelingLucky.tsx` | Discovery | Random article with weighted recency, popup card |

**Features implemented:**

| Feature | How |
|---------|-----|
| **Preferences persistence** | localStorage saves selected feeds, tags, sort order |
| **Reading history** | Tracks opened articles via title click; dims read articles |
| **Keyboard navigation** | `j`/`k` to navigate, `Enter` to open, `b` to bookmark |
| **Bookmarks** | Save/remove articles; `/feed/bookmarks` page with empty state |
| **Tag filtering** | Click tags to filter; multi-tag support |
| **Sort modes** | Newest (default) or HN-style ranked |
| **"I'm Feeling Lucky"** | Weighted random article selection |
| **Loading skeletons** | Animated pulse matching design system (`bg-border`) |
| **Empty/error states** | Clear filters button, retry on error |
| **Mobile tags** | Extra tag row on small screens |

**UI Design System Alignment:**
All components use `border-2 border-black` cards, `bg-accent` (purple) hover states, `bg-accent-yellow` calls-to-action, `text-[10px] font-bold uppercase tracking-wider` labels, `bg-surface-secondary` surfaces, `bg-border` loading states — consistent with the existing 100xSystems design system.

### 1.4 Navigation Integration

"Discover" link added to the main header navigation in `app/layout.tsx`.

### 1.5 Infrastructure

| Service | Use | Status |
|---------|-----|--------|
| **Vercel Hobby** (free) | Next.js hosting + API routes | ✅ Active |
| **Clerk** (50k MAU free) | User auth | ✅ Connected |
| **Turso** (5GB free) | Database | ✅ Connected (not yet used for feed) |
| **rss-parser** (npm) | RSS/Atom XML parsing | ✅ Installed |

---

## Part 2: Hard Problems & Solutions

> Research-backed analysis of the most significant challenges and how to solve them within our constraints (zero cost, serverless only, no VPS).
>
> **Note on effort estimates:** All estimates are for a single developer working solo. Timeline = effort × context-switching factor (~1.5x).

---

### Problem 1: Vercel 10-Second Function Timeout with 51 Feeds

**The problem:**
When a user selects all 51 feeds, the API route fetches RSS from all of them in parallel via `Promise.allSettled`. Each fetch takes 200ms–3s. In the worst case, a few slow feeds could push the total past Vercel Hobby's 10-second timeout limit. Vercel Hobby does NOT support Fluid Compute (extended timeouts).

**Research finding:** Attempting 50+ synchronous RSS fetches in a single Vercel function WILL eventually exceed the timeout.

**Solutions (ordered by complexity):**

**Solution 1A — Client-side progressive loading (Recommended for now)**
- Fetch only the first 10–15 feeds server-side, display them immediately
- Then have the client make additional API calls for remaining feeds in the background
- As articles arrive, they're appended to the feed (React state handles this naturally)
- Benefits: Instant perceived load time, no infrastructure changes
- Trade-off: More API calls from the client

**Solution 1B — Inngest/Trigger.dev fan-out (Medium-term)**
- Dispatch a single request to Inngest (free tier supports fan-out)
- Inngest spins up 51 isolated function calls, each fetching one feed
- Results are aggregated and stored
- Benefits: Reliable at any scale, built-in retries
- Trade-off: External dependency, learning curve

**Solution 1C — Feed batching (Manual approach)**
- Split 51 feeds into groups of 10
- Client makes sequential calls or the API processes in batches internally
- Benefits: Zero new dependencies
- Trade-off: More complex client logic

---

### Problem 2: Rate Limiting from Blog Platforms

**The problem:**
If the site gains traction, blog platforms (Medium, Ghost, WordPress.com, Substack) may rate-limit requests from Vercel's IP range. Each user request currently triggers fresh fetches from all selected feeds.

**Research finding:** The gold standard is HTTP Conditional GET using `If-Modified-Since` and `If-None-Match` headers. When a server returns `304 Not Modified`, the feed hasn't changed — skip re-parsing.

**Solutions:**

**Solution 2A — Conditional GET with Turso cache (Recommended)**
- On first fetch, store `ETag` and `Last-Modified` headers in Turso per-feed
- On subsequent requests, send `If-None-Match` and `If-Modified-Since`
- If server returns `304`, serve cached data from Turso
- If server returns `200`, update cache + ETag
- Benefits: 90% reduction in bandwidth, respects origin servers
- Implementation: ~50 lines of code in the API route

**Solution 2B — Turso response cache**
- Cache full RSS responses in Turso with a TTL (15–30 minutes)
- Serve from cache within the TTL; only re-fetch expired feeds
- Benefits: Zero repeated fetches within TTL
- Trade-off: Slightly stale data

**Solution 2C — Respect `Retry-After` + exponential backoff**
- Read `Retry-After` header from rate-limited responses
- Mark the feed as `cooling_down` in Turso until the retry time
- Benefits: Prevents hammering servers that are already overloaded

---

### Problem 3: No Full-Text Search

**The problem:**
With 51 feeds producing hundreds of articles, users need to search by keyword. Currently they can only filter by tags and feeds.

**Research finding:** Turso now supports Native FTS (built on Tantivy engine) — NOT the old SQLite FTS5 virtual table approach. Syntax: `CREATE INDEX idx_name ON table USING FTS (col1, col2)`. Supports BM25 ranking, transactional consistency. Free tier: 9GB storage, 500M row reads/month — more than enough.

**Solution — Turso Native FTS:**

> **⚠️ Ethical boundary:** We cache **searchable metadata only** — title, summary, author, feed name. We NEVER store the full article body, content, or images. This is strictly an index of metadata to enable search, not a content repository.

1. When API fetches articles, upsert metadata (title, summary, author, feed name) into a Turso table
2. Create an FTS index on that table
3. Expose `/api/feed/search?q=kubernetes` API endpoint
4. Client-side: `Fuse.js` for instant search of loaded articles, Turso FTS for full search

```sql
-- Table for searchable article metadata
CREATE TABLE article_search_cache (
  id TEXT PRIMARY KEY,
  title TEXT NOT NULL,
  summary TEXT,
  author TEXT,
  feed_name TEXT,
  feed_id TEXT NOT NULL,
  url TEXT NOT NULL,
  published_at TEXT,
  cached_at TEXT DEFAULT CURRENT_TIMESTAMP
);

-- Turso Native FTS index
CREATE INDEX idx_article_search ON article_search_cache USING FTS (title, summary, author, feed_name);
```

**Considerations:**
- Only cache metadata (title, summary), NOT full article content — ethical boundary maintained
- TTL-based expiration (re-fetch after 1 hour)
- Periodic `OPTIMIZE INDEX` to prevent search degradation

---

### Problem 4: Cross-Device Data Loss (localStorage Only)

**The problem:**
Bookmarks, reading history, and feed preferences are all in localStorage. Clearing browser data or switching devices loses everything. No cross-device sync.

**Solution — Turso sync with Clerk auth:**
- Clerk auth is already set up (50k MAU free tier)
- Create Turso tables for `bookmarks`, `reading_history`, `feed_preferences`
- On page load: if user is signed in, sync localStorage → Turso
- On preference change: write to Turso as well as localStorage
- localStorage stays as the fast cache; Turso is the persistent source of truth

```sql
CREATE TABLE user_bookmarks (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  user_id TEXT NOT NULL,
  article_url TEXT NOT NULL,
  title TEXT NOT NULL,
  feed_name TEXT NOT NULL,
  feed_id TEXT NOT NULL,
  saved_at TEXT DEFAULT CURRENT_TIMESTAMP,
  UNIQUE(user_id, article_url)
);

CREATE TABLE user_feed_preferences (
  user_id TEXT PRIMARY KEY,
  selected_feeds TEXT NOT NULL,  -- JSON array
  selected_tags TEXT NOT NULL,    -- JSON array
  sort_by TEXT NOT NULL DEFAULT 'newest'
);
```

**Sync strategy:**
1. User signs in → fetch Turso data → merge with localStorage → render
2. User changes preference → write to localStorage (instant) + POST to API (background)
3. API writes to Turso
4. On next device login → preferences are restored from Turso

---

### Problem 5: RSS Feed Breakage

**The problem:**
RSS feeds can change URLs, return 404s, or go silent. Currently no monitoring system.

**Research finding:** GitHub Actions is the cheapest option — scheduled workflow checks all feed URLs, reports 404s. Free tier: 2000 minutes/month, each check takes ~2 seconds × 51 feeds = ~2 minutes/run. Monthly cost: ~2 minutes × 4 runs = ~8 minutes.

**Solution — GitHub Actions health check:**
- Workflow runs weekly via `cron: '0 6 * * 1'` (Monday 6 AM)
- Script iterates all 51 feed URLs, checks for 200/304 status
- Reports failures to a GitHub Issue or email

**Pagination note:** The API should also add a `cursor` (publish date or article ID) query parameter. When fetching profiles, the API returns `cursor` in the response. The client passes it for the next page. This prevents overwhelming single requests with too many feeds and gives users incremental loading.

**Implementation (`.github/workflows/feed-health.yml`):**
```yaml
name: Feed Health Check
on:
  schedule:
    - cron: '0 6 * * 1'  # Every Monday 6 AM
  workflow_dispatch:  # Manual trigger

jobs:
  check-feeds:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Check feed URLs
        run: node scripts/check-feeds.mjs
```

---

### Problem 6: No Quality Signals (Upvotes)

**The problem:**
All articles start with 0 upvotes. The HN ranking algorithm (`(P-1)/(T+2)^G`) doesn't work without votes. No way to distinguish golden resources from mediocre posts.

**Solutions:**

**Solution 6A — Implicit signals (No auth needed)**
- Track article view counts via the API
- Track bookmark counts
- Track "I'm Feeling Lucky" click-through rates
- Use these as proxy quality signals

**Solution 6B — Explicit upvotes (Requires auth + Turso)**
- Add upvote/downvote buttons to ArticleCard
- Store votes per user in Turso
- Calculate weighted scores: `(upvotes - downvotes) / total_votes * recency`
- Display top articles with the HN ranking

**Solution 6C — Curator picks**
- Manually highlight exceptional articles with a "Staff Pick" badge
- Store curated article URLs in a JSON file or Turso
- These get bonus score in the ranking

---

### Problem 7: Duplicate Articles Across Feeds

**The problem:**
An article cross-posted to multiple blogs (e.g., an engineer who blogs at both their personal site and company blog) appears multiple times.

**Solution — URL deduplication:**
- In the API route, maintain a `Set<string>` of article URLs already seen
- Skip articles whose URL already exists in the set
- Client-side: same dedup logic on the articles array

**Implementation:**
```typescript
const seenUrls = new Set<string>();
const dedupedArticles = articles.filter(a => {
  if (seenUrls.has(a.url)) return false;
  seenUrls.add(a.url);
  return true;
});
```

---

### Problem 8: SEO — Feed Content Is Not Indexable

**The problem:**
The `/feed` page is entirely client-rendered (uses `'use client'` and fetches via API). Search engines see an empty page shell.

**Solutions:**

**Solution 8A — `/feed/[tag]` static pages (Recommended)**
- Generate static pages at build time for each tag (e.g., `/feed/distributed-systems`)
- In `generateStaticParams`, iterate `ALL_TAGS` and pre-fetch a few articles
- Page contains server-rendered article list + client hydration for interactivity
- Each tag page has unique meta tags and SEO-friendly content

**Solution 8B — Metadata-only SSG for `/feed`**
- Server component shell that renders article metadata from a cached source
- Only the bookmark/sort/filter interactivity is client-side
- Search engines see article titles and descriptions

---

### Problem 9: Vercel Bandwidth Limits (100 GB/month)

**The problem:**
Vercel Hobby has a 100 GB/month bandwidth cap. Each API response is ~50-100KB for 50 articles. At 100KB × 1M requests = 100 GB.

**Mitigations:**
- `Cache-Control: s-maxage=300` ensures edge cache serves identical requests (free bandwidth)
- Compress API responses (Vercel does gzip/brotli automatically)
- Pagination: default limit to 30 articles, load more on demand
- Smaller responses = less bandwidth
- Monitor Vercel dashboard for bandwidth usage

---

### Problem 10: No Email Digests / Notifications

**The problem:**
Users must manually visit `/feed` to see new articles. No way to get a daily/weekly digest.

**Solution — Vercel Cron + Resend (free tier):**
- Vercel Cron Job triggers daily at 8 AM
- Fetches articles from user's preferred feeds
- Generates a simple HTML digest email
- Sends via Resend (100 emails/day free tier)
- Users opt-in from their settings page

```typescript
// /api/cron/digest — triggered by Vercel Cron
export async function GET() {
  // Get all users with digest enabled from Turso
  // For each user, fetch their preferred feeds
  // Generate HTML email with top articles
  // Send via Resend API
}
```

---

## Part 3: What's Next — Prioritized Roadmap

### 🔴 Phase A: Scale & Reliability (0 infrastructure cost)

| Priority | Feature | Effort | Impact | Dependencies |
|----------|---------|--------|--------|-------------|
| **A1** | Client-side progressive loading — fetch 15 feeds first, rest in background | 2-3 days | High — solves timeout issue | None |
| **A2** | Conditional GET with Turso ETag cache *(needs API route restructure from parseURL to manual fetch+parseString)* | 3-4 days | High — reduces bandwidth & rate limits | Turso schema |
| **A3** | URL deduplication + pagination cursor in API | 1 day | Medium — removes duplicates, enables progressive loading | None |
| **A4** | GitHub Actions feed health check | 1 day | Medium — detects broken feeds | GitHub Actions |

### 🟡 Phase B: Search & Discovery (Turso integration)

| Priority | Feature | Effort | Impact | Dependencies |
|----------|---------|--------|--------|-------------|
| **B1** | `/api/feed/search?q=` with Turso Native FTS | 2 days | High — full-text search | A2, Turso schema |
| **B2** | `/feed/[tag]` static page generation | 2 days | High — SEO, shareable links | None |
| **B3** | Implicit quality signals (view count, bookmark count) | 1 day | Medium — better ranking | A2 |

### 🟢 Phase C: User Features (Clerk + Turso auth)

| Priority | Feature | Effort | Impact | Dependencies |
|----------|---------|--------|--------|-------------|
| **C1** | Upvote/downvote with Turso | 2-3 days | High — quality signals | Clerk auth |
| **C2** | Cross-device sync (bookmarks, preferences → Turso) | 3-4 days | High — data portability | Clerk auth, A2, A3 |
| **C3** | Curator picks / staff-picked articles | 1 day | Medium — trust signal | None |
| **C4** | Public collection pages (`/feed/collection/[id]`) | 3-4 days | Medium — community | C2 |

### 🔵 Phase D: Engagement & Growth

| Priority | Feature | Effort | Impact | Dependencies |
|----------|---------|--------|--------|-------------|
| **D1** | Community feed submissions (PR-based) | 2-3 days | Medium — feed growth | None |
| **D2** | Daily email digest via Resend *(requires: user prefs table, per-user feed fetch, HTML templates, opt-in UI, cron)* | 4-5 days | Medium — retention | C2 |
| **D3** | Article inline reading (via @mozilla/readability API) | 3-4 days | Medium — UX | A1 |
| **D4** | `/feed/random` page — always shows a random gem | 1 day | Low — delight | None |

---

## Part 4: Architectural Decisions Log

| Decision | Rationale | Date |
|----------|-----------|------|
| **JSON file for registry** (not Turso) | Static data, version-controlled, easy PRs, zero latency | July 23 |
| **Serverless API route as RSS proxy** | CORS restrictions on RSS feeds require server-side fetch | July 23 |
| **No article content stored** | Ethical — we don't own the content, only link to it | July 23 |
| **localStorage for MVP preferences** | Zero infrastructure cost, instant reads, no auth needed | July 23 |
| **`Promise.allSettled` with index-based error tracking** | Partial failure tolerance — one broken feed doesn't break all | July 23 |
| **`border-2 border-black` design pattern** | Matches existing SystemsListing component design | July 23 |
| **`@/` path alias maps to `./src/*`** | Existing project convention | July 23 |

---

## Part 5: Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Vercel removes Hobby free tier | Low | Critical | Migrate to Cloudflare Pages (free, unlimited bandwidth) |
| RSS feed URLs change | Medium | Medium | GitHub Actions health check, community reporting |
| Blog platform blocks Vercel IP | Low | Medium | Conditional GET reduces requests; rotate via different region |
| Turso free tier limits change | Low | Medium | Cache in localStorage primarily; Turso is secondary |
| User adoption exceeds bandwidth | Medium (6+ months) | Medium | Cloudflare Pages has unlimited bandwidth; migrate if needed |
| copyright/complaint from blog owner | Low | Low | We only link to content, never host it; DMCA-compliant |

---

*End of document. Last updated: July 23, 2026.*
