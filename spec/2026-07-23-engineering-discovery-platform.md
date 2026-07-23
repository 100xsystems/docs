# 100xSystems — Engineering Discovery Platform

> **Status:** Active Spec (July 23, 2026)
> **Author:** AI-assisted planning
> **Infrastructure constraint:** Zero cost, serverless only, no VPS

---

## 1. The Problem

Engineering knowledge is fragmented across thousands of blogs, architecture writeups, and technical deep-dives. The best content — from engineers at Stripe, Netflix, Discord, Cloudflare, and others — is invisible to most developers because:

- Google/AI can only answer what you ask — you can't search for what you don't know exists
- Hacker News is noisy and broad
- Lobste.rs is invite-only and small
- RSS readers require you to know which feeds to follow
- Newsletters are one-size-fits-all

**The pain:** Engineers miss golden resources that would make them better engineers, simply because they never knew those resources existed.

## 2. The Solution

**100xSystems becomes an Engineering Discovery Platform** — a curated directory of the best engineering blogs, where users can:

1. **Browse feeds** from top engineering blogs (Stripe, Netflix, Discord, Cloudflare, Martin Fowler, etc.)
2. **Filter by tags** (distributed-systems, databases, system-design, rust, etc.)
3. **Get a custom feed** — select which blogs to follow, get a unified timeline
4. **Discover hidden gems** — "I'm Feeling Lucky" surfaces random high-quality articles
5. **Collect and share** — bookmarks, reading lists, collections

## 3. Core Ethical Principle

**We do not own, store, or host any content.** We maintain a directory of **links** (RSS feed URLs). When a user visits, we fetch the RSS feeds on-the-fly, parse the metadata, and display it. The actual article content always lives on the original author's blog.

What we store:
- RSS feed URLs (public links, not content)
- User preferences (which feeds to follow — localStorage only for MVP)

What we NEVER store:
- Article content
- Article text
- Images or assets from articles
- Any copyrighted material

## 4. Architecture

### 4.1 Overview

```
┌─────────────────────────────────────────────────────────────────┐
│  FEED REGISTRY (JSON file)                                       │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  feeds.json                                              │    │
│  │  [                                                        │    │
│  │    { name, rssUrl, siteUrl, icon, tags, addedBy },        │    │
│  │    ... 100+ engineering blogs                              │    │
│  │  ]                                                        │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  USER VISITS /feed                                               │
│       │                                                          │
│       ▼                                                          │
│  Client loads feed registry JSON                                  │
│       │                                                          │
│       ▼                                                          │
│  User selects which blogs to follow (tag/feed toggles)            │
│       │                                                          │
│       ▼                                                          │
│  /api/feed?feeds=netflix,stripe,cloudflare                       │
│       │                                                          │
│       ▼                                                          │
│  Next.js Serverless Function:                                     │
│  ├─ Looks up feed URLs from registry                              │
│  ├─ Fetches RSS XML from each blog (Promise.all)                  │
│  ├─ Parses XML via rss-parser                                     │
│  └─ Returns JSON: { articles: [...] }                             │
│       │                                                           │
│       ▼                                                           │
│  Client renders the feed:                                          │
│  ├─ HN-style ranking: (P-1)/(T+2)^1.8                             │
│  ├─ "I'm Feeling Lucky" button                                    │
│  ├─ Bookmark articles (localStorage)                              │
│  └─ Click → open original article                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 Why Not a Database?

For the MVP, the feed registry is a **JSON file** because:
- It's static data (feed URLs don't change frequently)
- No database latency for reads
- Ships with the codebase, version-controlled
- Zero infrastructure cost
- Easy for community contributors to add new feeds via PR

**Future:** When we need community submissions, voting on feeds, or user-contributed feeds, migrate to Turso.

### 4.3 Why Serverless API Route for RSS Fetching?

RSS feed XML requests have **CORS restrictions** — most RSS XML endpoints do not set `Access-Control-Allow-Origin` headers, which means browsers block client-side `fetch()` calls. A serverless API route acts as a proxy:

1. Browser calls `/api/feed?feeds=netflix,stripe`
2. API route (running server-side) fetches RSS XML without CORS issues
3. API route parses XML, extracts article metadata
4. Returns JSON to browser

### 4.4 Content Flow

```
Request ──► Serverless Function ──► Fetch RSS feeds ──► Parse XML
                                                              │
                                                              ▼
                                                    Extract: title, url,
                                                    author, published date,
                                                    summary/snippet
                                                              │
                                                              ▼
                                                    Return JSON array
                                                    (no content stored)
                                                              │
                                                              ▼
                                                    Browser renders feed
                                                    User clicks → original URL
```

## 5. Feed Registry Format

### 5.1 JSON Structure

```json
[
  {
    "id": "netflix-tech-blog",
    "name": "Netflix Tech Blog",
    "rssUrl": "https://netflixtechblog.com/feed",
    "siteUrl": "https://netflixtechblog.com",
    "icon": "netflix",
    "tags": ["distributed-systems", "infrastructure", "streaming"],
    "description": "Engineering insights from Netflix's world-class infrastructure team",
    "addedBy": "curator",
    "language": "en"
  }
]
```

### 5.2 Initial Feed List (MVP)

| Blog | RSS URL | Tags |
|------|---------|------|
| Netflix Tech | `https://netflixtechblog.com/feed` | distributed-systems, infra |
| Stripe Engineering | `https://stripe.com/blog/feed.rss` | payments, architecture |
| Cloudflare Blog | `https://blog.cloudflare.com/rss` | networking, security |
| Discord Engineering | `https://discord.com/category/engineering/rss` | backend, infra |
| Uber Engineering | `https://www.uber.com/en-IN/blog/engineering/feed` | distributed-systems |
| Engineering at Meta | `https://engineering.fb.com/feed` | infrastructure, AI |
| Martin Fowler | `https://martinfowler.com/feed.atom` | architecture, patterns |
| AWS Architecture | `https://aws.amazon.com/blogs/architecture/feed` | cloud, architecture |
| Grafana Labs | `https://grafana.com/blog/index.xml` | observability |
| Apple ML Research | `https://machinelearning.apple.com/rss.xml` | ML, AI |
| Slack Engineering | `https://slack.engineering/feed` | backend, infra |
| Figma Engineering | `https://www.figma.com/blog/feed` | frontend, systems |
| Tailscale Blog | `https://tailscale.com/blog/index.xml` | networking, security |
| CockroachDB | `https://www.cockroachlabs.com/blog/index.xml` | databases |
| Svelte Blog | `https://svelte.dev/blog/rss.xml` | frontend |

## 6. API Route Design

### 6.1 `GET /api/feed`

**Query Parameters:**
| Param | Type | Description |
|-------|------|-------------|
| `feeds` | string | Comma-separated feed IDs (e.g., `netflix,stripe,cloudflare`) |
| `limit` | number | Max articles to return (default 50) |

**Response:**
```json
{
  "articles": [
    {
      "id": "netflix-tech-blog-2026-07-22-title-slug",
      "feedId": "netflix-tech-blog",
      "feedName": "Netflix Tech Blog",
      "title": "Scaling Cassandra at Netflix",
      "url": "https://netflixtechblog.com/scaling-cassandra",
      "author": "Netflix Engineering",
      "summary": "How Netflix scaled their Cassandra infrastructure...",
      "publishedAt": "2026-07-22T10:00:00.000Z",
      "tags": ["distributed-systems", "databases"],
      "contentSnippet": "First 200 characters of the article..."
    }
  ],
  "feeds": ["netflix-tech-blog", "stripe", "cloudflare"],
  "total": 3
}
```

### 6.2 No Authentication Required

All feed API endpoints are **unauthenticated** for the MVP. Anyone can:
- List available feeds
- Fetch articles from any feed
- Browse the feed page

Authentication (via Clerk) will be added later for:
- Saving custom feed configurations
- Bookmarks and reading lists
- Community submissions

## 7. Client-Side Architecture

### 7.1 React Components (in `src/feed/`)

```
src/feed/
├── feed.constants.ts    # Feed registry data
├── feed.types.ts        # TypeScript interfaces
├── feed.api.ts          # API call functions
├── feed.utils.ts        # HN ranking, sorting, filtering
├── FeedPage.tsx         # Main feed page client component
├── FeedSourceSelector.tsx  # Blog selection chips
├── ArticleCard.tsx      # Individual article display
├── FeedHeader.tsx       # Feed page header with filters
└── FeelingLucky.tsx     # Random article button
```

### 7.2 HN Ranking Algorithm

```typescript
function hnScore(points: number, hoursSincePublished: number, gravity = 1.8): number {
  return (points - 1) / Math.pow(hoursSincePublished + 2, gravity);
}
```

Since we don't have real upvotes initially, articles can be ranked by:
1. **Recency** (default) — newest first
2. **Combined score** — days since published + quality signals (future)
3. **"I'm Feeling Lucky"** — weighted random selection

### 7.3 localStorage Usage

```typescript
interface UserFeedPreferences {
  selectedFeeds: string[];    // Feed IDs user has selected
  bookmarks: Bookmark[];      // Saved articles { url, title, savedAt }
  readingHistory: string[];   // Article URLs visited
  hiddenFeeds: string[];      // Feeds user has dismissed
}
```

All stored in localStorage. No server calls needed for preferences.

## 8. Challenges & Limitations

### 8.1 Websites Without RSS Feeds

**The Problem:** Some engineering blogs don't publish RSS/Atom feeds.

**Solutions (future consideration, not MVP):**

| Approach | How It Works | Limitation |
|----------|-------------|------------|
| **RSSHub** | Open-source tool that generates RSS from many sites | Self-hosted = server needed. Public instances rate-limited |
| **@mozilla/readability** | Extracts content from HTML on-the-fly | Only works if we fetch the page content (CORS, rate limits) |
| **FreshRSS scraper** | Monitors HTML pages for changes | Requires a server |
| **Manual feed creation** | We subscribe and manually verify | High maintenance |
| **Community submitted feeds** | Users submit feed URLs via PR | Requires trust and moderation |

**Impact on MVP:** We **start with blogs that HAVE RSS feeds** (most engineering blogs do — Netflix, Stripe, Cloudflare, Discord, Uber, Meta, etc.). Sites without RSS are a future concern.

### 8.2 Platform-Specific Notes

| Platform | RSS Support | Notes |
|----------|------------|-------|
| **Medium** | ✅ `medium.com/feed/@user` or `medium.com/feed/publication` | Works for both users and publications |
| **GitHub** | ✅ `repo/releases.atom`, `repo/commits.atom` | Only Atom, not RSS |
| **Substack** | ✅ Auto-generates RSS | `substack.com/feed` |
| **Ghost** | ✅ Auto-generates RSS | `blog.com/rss` |
| **WordPress** | ✅ Auto-generates RSS | `blog.com/feed` |
| **Hashnode** | ✅ Auto-generates RSS | `blog.com/rss.xml` |
| **Google Research** | ⚠️ Blog has no published feed | Needs RSSHub or scraping |
| **Figma Blog** | ✅ Has RSS at `/blog/feed` | Works |

### 8.3 Vercel Hobby Plan Limitations

| Limit | Value | Impact |
|-------|-------|--------|
| Function timeout | 10 seconds | Need to limit feeds per request or batch |
| Bandwidth | 100 GB/month | ~200k page views/month with avg 500KB |
| Build minutes | 6000/month | Only triggered on code changes (not daily) |
| Concurrent functions | Managed | Good enough for feed fetching |

**Mitigation:** The feed API fetches RSS feeds in parallel using `Promise.all`. At ~200ms-1s per feed fetch, we can comfortably handle 10-20 feeds in a single request within the 10s limit. More feeds = more requests (pagination).

### 8.4 CORS

**RSS feeds do NOT set CORS headers.** This is why a serverless API route is required — browser `fetch()` calls to `https://netflixtechblog.com/feed` would be blocked by CORS. The Next.js API route runs server-side and has no CORS restrictions.

### 8.5 Rate Limiting

Some blog platforms may rate-limit frequent requests. Mitigations:
- `Cache-Control` headers on API responses (browser caches for 5 minutes)
- No per-user polling — all users share the same cached response
- Future: Turso cache with TTL

## 9. Libraries Used

| Library | Version | Purpose | Why |
|---------|---------|---------|-----|
| `rss-parser` | latest | Parse RSS/Atom XML | Battle-tested, handles all feed formats |
| `react-virtuoso` | latest | Virtual scrolling | Handles 1000+ articles smoothly |
| `Fuse.js` | latest | Client-side search | Fuzzy search across loaded articles |
| `react-hotkeys-hook` | latest | Keyboard navigation | j/k for feed browsing |
| `@dnd-kit` | latest | Drag-and-drop collections (future) | |
| Custom HN rank | — | Article ranking | Simple math, no ML needed |

**All libraries are open-source, free, and well-maintained.**

## 10. Implementation Order

### Phase 1 — Foundation (Current Sprint)
1. Create feed registry JSON with 15+ engineering blogs
2. Create `/api/feed` serverless API route — fetch RSS, parse, return JSON
3. Test API with `curl` — verify it returns real articles
4. Create `/feed` page — basic feed display with client-side rendering

### Phase 2 — Discovery Features (Next Sprint)
5. Feed source selector — toggle which blogs to follow
6. Tag-based filtering
7. "I'm Feeling Lucky" random article
8. HN-style ranking with temporal decay

### Phase 3 — Social Features (Future)
9. Bookmark articles (localStorage)
10. Custom feed configuration persistence
11. Community feed submissions
12. Reading history → personalized recommendations

## 11. File Structure (New, Isolated)

```
website/
├── app/
│   ├── feed/                    # NEW — Feed route
│   │   └── page.tsx             # Feed page (server component shell)
│   └── api/
│       └── feed/
│           └── route.ts         # NEW — Feed API (serverless)
├── src/
│   └── feed/                    # NEW — All feed-related code
│       ├── feed.constants.ts    # Feed registry (JSON data)
│       ├── feed.types.ts        # TypeScript interfaces
│       ├── feed.api.ts          # API call functions
│       ├── feed.utils.ts        # HN ranking, helpers
│       ├── FeedPage.tsx         # Main feed client component
│       ├── FeedSourceSelector.tsx  # Blog selection UI
│       ├── ArticleCard.tsx      # Article display card
│       ├── FeedHeader.tsx       # Header with filters
│       └── FeelingLucky.tsx     # Random article feature
└── docs/
    └── spec/
        └── 2026-07-23-engineering-discovery-platform.md  # This doc
```

## 12. Future Considerations

- **Community curation**: Users submit new feeds via PR (like awesome lists)
- **Upvoting**: Article upvotes (Turso + auth required)
- **Collections**: Drag-and-drop article collections
- **Daily digest**: Email of top articles (Vercel Cron → free email API)
- **SEO**: Each feed/tag combination becomes a discoverable page
- **Migration to DB**: When feed registry grows large or needs community editing

---

*End of specification.*
