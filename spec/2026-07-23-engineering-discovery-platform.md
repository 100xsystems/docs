# Engineering Discovery Platform — Complete Specification

> **Date:** 2026-07-23
> **Status:** Active Development
> **Repos:** `website` (Next.js), `registry` (data pipeline)
> **Stack:** Next.js (App Router), Turso (libSQL), Clerk (Auth), Vercel (Deploy), GitHub Actions (Cron)

---

## Table of Contents

1. [Vision & Philosophy](#1-vision--philosophy)
2. [Architecture Overview](#2-architecture-overview)
3. [Feature: RSS Feed System](#3-feature-rss-feed-system)
4. [Feature: Knowledge Graph](#4-feature-knowledge-graph)
5. [Feature: Awesome Lists](#5-feature-awesome-lists)
6. [Feature: Bookmark Sync (Turso)](#6-feature-bookmark-sync-turso)
7. [Feature: Feed Preference Sync (Turso)](#7-feature-feed-preference-sync-turso)
8. [Feature: Search Term Highlighting](#8-feature-search-term-highlighting)
9. [Data Pipeline (Registry)](#9-data-pipeline-registry)
10. [Website Caching Strategy](#10-website-caching-strategy)
11. [Navigation & UX](#11-navigation--ux)
12. [What's Next](#12-whats-next)
13. [Appendix: File Map](#13-appendix-file-map)

---

## 1. Vision & Philosophy

### The Problem

Engineering knowledge exists but is fragmented, undiscoverable, and impossible to navigate:
- Engineering blogs are isolated and hard to discover
- Awesome lists exist but are buried in GitHub
- Engineering concepts are scattered across Wikipedia, blog posts, and documentation
- Search assumes you know what to search for; AI assumes you know what to ask
- No platform connects the dots between concepts, implementations, and resources

### The Mission

> Help engineers discover the knowledge, projects, people, and ideas they didn't know they needed.

### Core Principles

| Principle | Description |
|-----------|-------------|
| **Metadata only** | Never store article body content. Always link to the original source. |
| **Static-first** | Data is pre-processed at the registry level; the website reads static files. |
| **Local-first** | User preferences and bookmarks sync through localStorage first, then persist to DB. |
| **Free infrastructure** | Vercel Hobby, Turso free tier, GitHub Actions (all free). |
| **Incremental updates** | Registry runs daily via GitHub Actions cron; website uses ISR + delta files. |

---

## 2. Architecture Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                        REGISTRY REPO                             │
│  (Data pipeline — runs locally or via GitHub Actions cron)       │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐   │
│  │ RSS Updater  │  │ Awesome List │  │ Knowledge Graph     │   │
│  │ update-feeds │  │ Crawler      │  │ Crawler             │   │
│  │ .ts          │  │ crawl-awesome│  │ crawl-knowledge.ts  │   │
│  │              │  │ .ts          │  │                      │   │
│  └──────┬───────┘  └──────┬───────┘  └──────────┬───────────┘   │
│         │                 │                      │               │
│         ▼                 ▼                      ▼               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐   │
│  │ feeds/       │  │ awesome/     │  │ knowledge/           │   │
│  │ {id}.json    │  │ {repo}.json  │  │ {category}/{slug}.   │   │
│  │ daily/delta  │  │ index.json   │  │ json                 │   │
│  │ .json        │  │              │  │ manifest.json        │   │
│  └──────────────┘  └──────────────┘  └──────────────────────┘   │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
        │                        │                      │
        │ git push               │ git push             │ git push
        ▼                        ▼                      ▼
┌──────────────────────────────────────────────────────────────────┐
│                       WEBSITE (Next.js)                          │
│                                                                  │
│  ┌────────────────────────────── CLIENT ──────────────────────┐  │
│  │  /feed        — RSS articles with Fuse.js search + filters │  │
│  │  /feed/[tag]  — Pre-filtered tag pages (SEO-friendly)      │  │
│  │  /feed/bookmarks — Cross-device bookmarks                  │  │
│  │  /discover/awesome — Flat awesome list feed                │  │
│  │  /knowledge   — Browse concepts by category                │  │
│  │  /knowledge/[concept] — Individual concept pages           │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌─────────────────────── BUILD CACHE ───────────────────────┐  │
│  │  public/feed-cache.json         — Combined RSS feed data  │  │
│  │  public/awesome-cache/          — Awesome list JSONs      │  │
│  │  public/knowledge-cache/        — Knowledge entities      │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌─────────────────────── PERSISTENCE ───────────────────────┐  │
│  │  Turso: feed_bookmarks, feed_preferences                  │  │
│  │  Clerk: Authentication (user IDs)                         │  │
│  │  localStorage: Bookmark cache, preference cache, reading  │  │
│  │               history, search state                       │  │
│  └────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

### Data Flow

```
Registry (crawl) ──git push──► GitHub ──fetch cache──► Website Build (ISR)
     │                            │                           │
     │ (daily cron)               │ (clone raw JSON)          │ (read static)
     ▼                            ▼                           ▼
  Updated Registry         Vercel Build                    Browser
  JSON files              public/ cache                    User sees content
```

---

## 3. Feature: RSS Feed System

### Status: ✅ COMPLETE

### Files

| File | Purpose |
|------|---------|
| `src/feed/feed.constants.ts` | Registry of 51 engineering blogs with RSS URLs and tags |
| `src/feed/feed.types.ts` | TypeScript interfaces for feed data |
| `src/feed/feed.cache.ts` | Cache loading logic with delta optimization |
| `src/feed/feed.api.ts` | Client-side API helpers for fetching feed data |
| `src/feed/feed.utils.tsx` | Sort, filter, highlight, random article helpers |
| `src/feed/FeedPage.tsx` | Main feed client component |
| `src/feed/FeedHeader.tsx` | Feed header with controls, tag filter chips |
| `src/feed/FeedSourceSelector.tsx` | Multi-select dropdown for feed sources |
| `src/feed/FeelLucky.tsx` | "I'm Feeling Lucky" random article button |
| `src/feed/ArticleCard.tsx` | Article display card with bookmark, share, open |
| `app/feed/page.tsx` | Feed route |
| `app/feed/[tag]/page.tsx` | Tag-filtered feed routes (SEO) |
| `app/feed/bookmarks/page.tsx` | Bookmark listing page |
| `app/api/feed/route.ts` | API endpoint serving cached feed data |

### Key Features

- **51 engineering blogs** curated from top companies (Netflix, Stripe, Cloudflare, Discord, etc.)
- **Fuse.js fuzzy search** on title, summary, author, feed name, tags
- **Keyboard navigation**: `j`/`k` to navigate, `Enter` to open, `b` to bookmark
- **Tag filtering**: Click tag chips to filter by topic
- **Source filtering**: Multi-select feed source dropdown
- **Sort modes**: Newest first or HN-style ranking
- **"I'm Feeling Lucky"**: Weighted random article selector
- **Reading history**: Tracks read articles (opacity dims read ones)
- **ISR + Delta optimization**: Fetches only new items from daily delta.json
- **Per-tag SEO pages**: `/feed/distributed-systems`, `/feed/databases`, etc.

---

## 4. Feature: Knowledge Graph

### Status: ✅ COMPLETE

### Files

| File | Purpose |
|------|---------|
| `registry/knowledge/seeds.json` | 162 seed concepts across 4 categories |
| `registry/scripts/crawl-knowledge.ts` | Hybrid Wikipedia + Wikidata crawler |
| `website/src/knowledge/knowledge.types.ts` | Knowledge entity & manifest types |
| `website/scripts/fetch-knowledge-cache.mjs` | Build-time cache script |
| `website/app/knowledge/page.tsx` | Knowledge browse page (server) |
| `website/app/knowledge/KnowledgeGraphClient.tsx` | Client search + highlight component |
| `website/app/knowledge/[concept]/page.tsx` | Individual concept detail page |

### Data Sources

| Source | Provides | Coverage |
|--------|----------|----------|
| **Wikipedia REST API** | Clean plain-text summary + description | ~90% of concepts |
| **Seed data** | Fallback descriptions for niche concepts | ~10% (YAGNI, KISS, etc.) |
| **QID references** | External Wikidata links | Included where available |

### Entity Schema

```json
{
  "id": "acid",
  "category": "principles",
  "label": "ACID",
  "description": "Set of properties guaranteeing reliable database transactions",
  "summary": "In computer science, ACID is a set of properties...",
  "parents": [],
  "children": [],
  "related": [],
  "externalUrls": {
    "wikipedia": "https://en.wikipedia.org/wiki/ACID",
    "wikidata": "https://www.wikidata.org/wiki/Q215616"
  }
}
```

### Concepts by Category

| Category | Count | Examples |
|----------|-------|----------|
| Principles | ~34 | ACID, CAP Theorem, SOLID, YAGNI, DRY, KISS |
| Languages | ~28 | Rust, Go, TypeScript, Python, Java, C++ |
| Tools | ~56 | Kubernetes, Docker, PostgreSQL, React, Kafka |
| Patterns | ~44 | Singleton, CQRS, Paxos, Raft, MapReduce |
| **Total** | **~162** | |

### Routes

| Route | Type | Description |
|-------|------|-------------|
| `/knowledge` | SSG + ISR (24h) | Browse all concepts by category |
| `/knowledge/[concept]` | SSG (pre-rendered) | Individual concept detail page |

### Search

- Client-side Fuse.js fuzzy search on concept label, slug, and category
- Category filter via URL params (`?category=principles`)
- Matched terms highlighted with `<mark>` styling

---

## 5. Feature: Awesome Lists

### Status: ✅ COMPLETE

### Files

| File | Purpose |
|------|---------|
| `registry/scripts/crawl-awesome.ts` | GitHub Awesome list README parser |
| `website/scripts/fetch-awesome-cache.mjs` | Build-time cache script |
| `website/app/discover/awesome/page.tsx` | Server component (loads all list JSONs) |
| `website/app/discover/awesome/AwesomeFeed.tsx` | Client component with search + filter |

### Crawled Resources

| Metric | Count |
|--------|-------|
| Awesome lists | **56** |
| Total unique resources | **~29,119** |
| Categories covered | **11** (Systems, DevOps, Languages, Databases, Architecture, Security, AI/ML, Observability, Testing, Tools, Emerging) |

### README Parser Capabilities

The `crawl-awesome.ts` script handles:
- **ATX headings** (`## Section`, `### Section`, etc.)
- **Setext headings** (text followed by `---` or `===`)
- **Hyphen lists** (`- [Title](url) - Description`)
- **Asterisk lists** (`* [Title](url)`)
- **Ordered lists** (`1. [Title](url)`)
- **Markdown table cells** (`| [Title](url) | Description |`)
- **HTML `<a>` tags** in table cells
- **Bare markdown links** (`[Title](url)`)
- **Nested list trimming** (only top-level links are captured)
- **TOC/Contributing/License section skipping**
- **URL deduplication** (normalized by hostname + path)

### CLI Options

```bash
# Crawl all predefined repos
GITHUB_TOKEN=ghp_xxx npm run crawl-awesome

# Dry run
npm run crawl-awesome -- --dry-run

# Debug mode (prints first 60 lines of each README)
npm run crawl-awesome -- --debug

# Single list
npm run crawl-awesome -- --list=awesome-distributed-systems

# Auto-discover by topic
npm run crawl-awesome -- --discover --topic=distributed-systems
```

### Website UI

- **Flat single-page feed** (no nested per-list routes)
- **Violet/purple theme** (filters, badges, empty state)
- **Fuse.js fuzzy search** on title, description, category, source
- **Source filter chips** (color-coded by awesome list)
- **Category filter** (click any of 30+ category tags)
- **Pagination**: Shows 100 items at a time with "Show more" button
- **Search term highlighting**: Matched terms highlighted in titles and descriptions

---

## 6. Feature: Bookmark Sync (Turso)

### Status: ✅ COMPLETE

### Files

| File | Purpose |
|------|---------|
| `supabase/migrations/20260723_feed_bookmarks.sql` | Migration SQL |
| `src/lib/db.ts` | DB helpers: `addFeedBookmark`, `removeFeedBookmark`, `getFeedBookmarks`, `upsertFeedBookmarks`, `clearFeedBookmarks` |
| `app/api/feed/bookmarks/route.ts` | REST API: GET, POST, DELETE |
| `src/feed/useBookmarks.ts` | Client hook: localStorage-first, Turso sync |

### Table Schema

```sql
CREATE TABLE IF NOT EXISTS feed_bookmarks (
    user_email TEXT NOT NULL,
    url TEXT NOT NULL,
    title TEXT NOT NULL,
    feed_name TEXT NOT NULL DEFAULT '',
    feed_id TEXT NOT NULL DEFAULT '',
    saved_at TEXT NOT NULL DEFAULT (datetime('now')),
    PRIMARY KEY (user_email, url)
);
```

### Sync Strategy

```
localStorage-first (instant render)
        │
        ▼
    Background Turso sync (when signed in)
        │
        ▼
    Server wins on merge (source of truth)
        │
        ▼
    Revert localStorage on API error
```

### Sync Events

| Event | Action |
|-------|--------|
| **On mount (signed in)** | Load localStorage → Fetch from Turso → Merge → Update localStorage |
| **Add bookmark** | Update localStorage immediately → POST to Turso → Revert on error |
| **Remove bookmark** | Update localStorage immediately → DELETE from Turso → Revert on error |
| **Sign in** | Load localStorage → Fetch from Turso → If local has more, upload all to Turso |
| **Clear all** | Clear localStorage → DELETE all from Turso → Revert on error |

---

## 7. Feature: Feed Preference Sync (Turso)

### Status: ✅ COMPLETE

### Files

| File | Purpose |
|------|---------|
| `supabase/migrations/20260723_feed_preferences.sql` | Migration SQL |
| `src/lib/db.ts` | DB helpers: `getFeedPreferences`, `upsertFeedPreferences`, `deleteFeedPreferences` |
| `app/api/feed/preferences/route.ts` | REST API: GET, POST, DELETE |
| `src/feed/useFeedPreferences.ts` | Client hook: localStorage-first, Turso sync |

### Table Schema

```sql
CREATE TABLE IF NOT EXISTS feed_preferences (
    user_email TEXT NOT NULL PRIMARY KEY,
    selected_feeds TEXT NOT NULL DEFAULT '[]',
    selected_tags TEXT NOT NULL DEFAULT '[]',
    sort_by TEXT NOT NULL DEFAULT 'newest',
    updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);
```

### Sync Strategy

Same localStorage-first + Turso sync pattern as bookmarks:
- **Single-row-per-user** (PK = user_email)
- Arrays stored as JSON strings in TEXT columns
- Server-wins merge for feeds/tags
- Local non-default sortBy preference preserved across devices
- Debounced sync (skips API call if serialized value unchanged)

### Migration to Apply

```bash
turso db shell <db-name> < supabase/migrations/20260723_feed_preferences.sql
```

---

## 8. Feature: Search Term Highlighting

### Status: ✅ COMPLETE

### Files

| File | Purpose |
|------|---------|
| `src/feed/feed.utils.tsx` | Shared `highlightMatches(text, query)` utility |
| `app/discover/awesome/AwesomeFeed.tsx` | Uses highlight on title + description |
| `src/feed/ArticleCard.tsx` | Uses highlight on title + summary |
| `src/feed/FeedPage.tsx` | Passes `searchQuery` to ArticleCard |
| `app/knowledge/KnowledgeGraphClient.tsx` | Uses highlight on concept labels |

### Implementation

```typescript
function highlightMatches(text: string, query: string): React.ReactNode
```

- Case-insensitive regex matching
- Wraps matched portions in `<mark>` with contextual color
- `.filter(Boolean)` prevents empty `<span>` elements from `split()`
- Returns plain text when no active search (zero overhead)
- Imported from shared utility in `feed.utils.tsx`

### Highlight Colors by Page

| Page | Color | Rationale |
|------|-------|-----------|
| Feed | `bg-amber-200 text-amber-900` | Contrasts with purple accent |
| Awesome Lists | `bg-violet-200 text-violet-900` | Matches purple theme |
| Knowledge Graph | `bg-neutral-200 text-neutral-900` | Matches neutral theme |

---

## 9. Data Pipeline (Registry)

### Status: ✅ COMPLETE

### Files

| File | Purpose |
|------|---------|
| `registry/scripts/update-feeds.ts` | RSS feed fetcher + delta generator |
| `registry/scripts/feed-registry.ts` | Feed source definitions (shared with website) |
| `registry/scripts/crawl-awesome.ts` | GitHub Awesome list README parser |
| `registry/scripts/crawl-knowledge.ts` | Hybrid Wikipedia + Wikidata crawler |
| `registry/scripts/crawl-sitemap.ts` | Sitemap-based historical import |

### Registry Scripts

```bash
# RSS — daily incremental update
npm run update-feeds              # Fetches latest 50 items per feed
npm run update-feeds -- --historical --limit=500   # One-time historical import
npm run update-feeds -- --feed=netflix-tech-blog   # Single feed

# Awesome Lists — GitHub README crawler
npm run crawl-awesome             # Crawl all 56 predefined lists
npm run crawl-awesome -- --dry-run
npm run crawl-awesome -- --debug

# Knowledge Graph — Wikipedia hybrid
npm run crawl-knowledge           # Enrich 162 concepts with Wikipedia summaries
npm run crawl-knowledge -- --dry-run

# Sitemap — historical blog article discovery
npm run crawl-sitemap             # Discover old articles from sitemaps
```

### GitHub Actions Cron (Planned)

The registry repository has a scheduled GitHub Action that runs `update-feeds.ts` daily:

```yaml
on:
  schedule:
    - cron: '0 0 * * *'   # Every day at 00:00 UTC
  workflow_dispatch:       # Manual trigger for testing
```

The workflow:
1. Checks out the registry repository
2. Runs `npm run update-feeds`
3. If JSON files changed, commits and pushes back to the repository

This keeps the feed data fresh without any external cron service.

---

## 10. Website Caching Strategy

### Status: ✅ COMPLETE

### Files

| File | Purpose |
|------|---------|
| `scripts/fetch-feed-cache.mjs` | Build-time feed cache (copy or clone from registry) |
| `scripts/fetch-knowledge-cache.mjs` | Build-time knowledge cache |
| `scripts/fetch-awesome-cache.mjs` | Build-time awesome list cache |
| `src/feed/feed.cache.ts` | Runtime cache loading (ISR delta merging) |

### Cache Hierarchy (Runtime — feed.cache.ts)

```
1. /tmp/feed-cache.json          (ISR revalidation output — fastest)
   │   Is this fresh (< 24h)?
   ├── Yes → Return immediately
   └── No  → Fetch daily/delta.json from registry
              ├── New items? → Merge into /tmp/, return
              └── No items?  → Touch /tmp/ (reset staleness), return
              └── Fail?      → Use stale /tmp/ (still valid data)

2. public/feed-cache.json        (Build-time artifact)
   │   Is this fresh (< 24h)?
   ├── Yes → Return immediately
   └── No → Is it < 72h old?
       ├── Yes → Try delta merge
       └── No → Full re-fetch from GitHub raw

3. Full fetch from GitHub raw    (Cold start / very stale)
   └── Fetch all 51 feed JSON files in parallel
```

### Cache Scripts (Build-time)

Each cache script has two strategies:
- **Strategy A**: Copy from local filesystem (development, instant)
- **Strategy B**: Shallow clone the registry repo from GitHub (CI/build)

```bash
# Runs automatically during `next build` via prebuild scripts
node scripts/fetch-feed-cache.mjs
node scripts/fetch-knowledge-cache.mjs
node scripts/fetch-awesome-cache.mjs
```

### ISR Configuration

```typescript
// Knowledge Graph — pages revalidate every 24 hours
export const revalidate = 86400;

// Feed — revalidates via /api/feed with Cache-Control headers
// The API checks /tmp/feed-cache.json freshness
```

---

## 11. Navigation & UX

### Status: ✅ COMPLETE

### Header Navigation

The navigation in `app/layout.tsx` has:

```
[Systems]  [Knowledge Base]  [Discover]  [CLI Docs]  [Search]
              ├─ Principles     ├─ Feed
              ├─ Patterns       ├─ Awesome Lists
              ├─ Tools          └─ Knowledge Graph
              └─ Technologies
```

### Shared UI Patterns

| Pattern | How |
|---------|-----|
| **Fuse.js search** | Consistent across Feed, Awesome Lists, Knowledge Graph — same threshold, distance, lazy-init pattern |
| **Keyboard nav** | Feed page: `j`/`k` navigate, `Enter` open, `b` bookmark |
| **Tag chips** | Consistent filter chips across Feed (purple accent) and Knowledge Base |
| **Highlighting** | `highlightMatches()` shared utility in `feed.utils.tsx` |
| **Bookmarks** | localStorage-first + Turso sync, bookmark icon persistent |
| **localStorage-first** | All user data renders instantly from cache, syncs in background |
| **ISR + Stale-while-revalidate** | Data can be slightly stale for instant UX |

---

## 12. What's Next

### Priority: HIGH

| Feature | Description | Effort |
|---------|-------------|--------|
| **Unified `/discover` landing page** | Combined landing page with cards for Feed, Awesome Lists, and Knowledge Graph — current links go directly to each sub-page | 1 day |
| **Reading history persistence** | Currently localStorage-only. Add Turso table + sync (same pattern as bookmarks) | 1 day |
| **GitHub Actions cron setup** | Configure scheduled workflow in the registry repo for daily feed updates | 0.5 day |
| **Auto-apply migrations** | Create a setup script that auto-runs both SQL migrations | 0.5 day |

### Priority: MEDIUM

| Feature | Description | Effort |
|---------|-------------|--------|
| **Medium integration** | Crawl Medium publications by topic (uses Medium RSS by user ID) | 2 days |
| **Reddit integration** | Aggregate top engineering posts from subreddits via RSS | 1 day |
| **YouTube integration** | Curate engineering conference talks and channels | 2 days |
| **Hacker News integration** | Fetch top HN posts related to engineering topics | 1 day |
| **Wikipedia relationships** | Populate `parents`/`children`/`related` arrays via Wikidata SPARQL | 1 day |
| **Knowledge graph aliases** | Add common aliases to seed data (e.g., "Singleton Pattern") | 0.5 day |
| **Keyboard nav on Awesome/Knowledge** | Add `j`/`k`/`Enter` navigation to all list pages | 1 day |
| **Shared Fuse.js utility hook** | Extract common Fuse.js init + search pattern into `useSearch` hook | 0.5 day |
| **Dark mode** | Add CSS variables + toggle for the discovery pages | 2 days |
| **Mobile feed improvements** | Swipe gestures, bottom sheet filter, sticky search | 2 days |

### Priority: LOW

| Feature | Description | Effort |
|---------|-------------|--------|
| **More awesome lists** | Expand from 56 → 100+ lists (find niche topics) | 1 day |
| **More RSS feeds** | Expand from 51 → 75+ engineering blogs | 1 day |
| **Knowledge graph growth** | Add more concepts to seeds.json (e.g., ADR, Event Sourcing, C4 model) | 1 day |
| **Community contributions** | Allow users to submit new feed sources via PR template | 2 days |
| **Reading time estimation** | Show estimated reading time on ArticleCards | 0.5 day |
| **Related articles** | Suggest articles by same tag/feed on /knowledge pages | 1 day |
| **Social share** | More share targets (Twitter, LinkedIn, HN) beyond native share | 0.5 day |
| **Podcast feed support** | Add support for audio/video feeds alongside RSS articles | 2 days |
| **Analytics** | Track which feeds/topics are most engaged (privacy-first) | 2 days |
| **CLI integration** | Add a `discover` command to the CLI for terminal-based browsing | 3 days |

### Known Issues

| Issue | Status | Notes |
|-------|--------|-------|
| SortBy merge bias with multiple devices | Minor | `useFeedPreferences` has a subtle bias toward local non-default sort choices |
| Feed API token limitation | Info | Clerk auth used; no anonymous read-backup for non-signed-in bookmark sync |
| Build time degradation | Info | 51 feeds × 10 items + 56 awesome lists = cache generation takes ~15s on build |
| `/tmp/` cache persistence on Vercel | Info | Vercel serverless may clear `/tmp/` between ISR revalidations — delta fallback handles this |
| `.ts` → `.tsx` rename | ✅ Fixed | `feed.utils.ts` renamed to `feed.utils.tsx` for JSX support |

---

## 13. Appendix: File Map

### Registry (`/registry/`)

```
registry/
├── package.json                      # Scripts: update-feeds, crawl-*, etc.
├── feeds/                            # Generated RSS feed JSON files
│   ├── netflix-tech-blog.json
│   ├── stripe-engineering.json
│   └── ... (51 feeds)
├── daily/
│   └── delta.json                    # Daily new items (website ISR reads this)
├── awesome/                          # Generated awesome list JSON files
│   ├── index.json                    # Aggregated index (56 lists, 29k links)
│   ├── sindresorhus-awesome.json
│   └── ... (55 more lists)
├── knowledge/                        # Knowledge graph entities
│   ├── seeds.json                    # 162 seed concepts
│   ├── manifest.json                 # labelMap + categoryMap
│   ├── principles/
│   │   ├── acid.json
│   │   ├── cap-theorem.json
│   │   └── ... (34 concepts)
│   ├── languages/
│   │   ├── rust.json
│   │   ├── go.json
│   │   └── ... (28 concepts)
│   ├── tools/
│   │   ├── kubernetes.json
│   │   ├── docker.json
│   │   └── ... (56 concepts)
│   └── patterns/
│       ├── singleton.json
│       ├── cqrs.json
│       └── ... (44 concepts)
└── scripts/
    ├── update-feeds.ts               # RSS feed updater
    ├── feed-registry.ts              # Feed source definitions
    ├── crawl-awesome.ts              # GitHub Awesome list crawler
    ├── crawl-knowledge.ts            # Wikipedia + Wikidata hybrid
    └── crawl-sitemap.ts              # Sitemap-based historical import
```

### Website (`/website/`)

```
website/
├── app/
│   ├── layout.tsx                    # Root layout with navigation
│   ├── feed/
│   │   ├── page.tsx                  # Feed route
│   │   ├── [tag]/page.tsx            # Tag-filtered feed routes (SEO)
│   │   └── bookmarks/page.tsx        # Bookmark listing
│   ├── knowledge/
│   │   ├── page.tsx                  # Knowledge browse (server)
│   │   ├── KnowledgeGraphClient.tsx  # Knowledge search (client)
│   │   └── [concept]/page.tsx        # Individual concept pages
│   ├── discover/
│   │   └── awesome/
│   │       ├── page.tsx              # Awesome list server component
│   │       └── AwesomeFeed.tsx       # Awesome list client component
│   └── api/
│       └── feed/
│           ├── route.ts              # Feed data API
│           ├── bookmarks/route.ts    # Bookmark CRUD API
│           └── preferences/route.ts  # Preference CRUD API
├── src/
│   ├── feed/
│   │   ├── feed.constants.ts         # Source registry (51 blogs)
│   │   ├── feed.types.ts             # TypeScript interfaces
│   │   ├── feed.utils.tsx            # Shared utilities (sort, filter, highlight)
│   │   ├── feed.cache.ts             # Cache loading logic
│   │   ├── feed.api.ts               # API helpers
│   │   ├── FeedPage.tsx              # Main feed component
│   │   ├── FeedHeader.tsx            # Header with controls
│   │   ├── FeedSourceSelector.tsx    # Source multi-select
│   │   ├── ArticleCard.tsx           # Article display card
│   │   ├── FeelingLucky.tsx          # Random article button
│   │   ├── useBookmarks.ts           # Bookmark hook with Turso sync
│   │   └── useFeedPreferences.ts     # Preference hook with Turso sync
│   ├── knowledge/
│   │   └── knowledge.types.ts        # Knowledge entity types
│   └── lib/
│       └── db.ts                     # Turso client + all DB helpers
├── scripts/
│   ├── fetch-feed-cache.mjs          # Build-time feed cache
│   ├── fetch-knowledge-cache.mjs     # Build-time knowledge cache
│   └── fetch-awesome-cache.mjs       # Build-time awesome list cache
└── supabase/migrations/              # SQL migrations
    ├── 20260723_feed_bookmarks.sql   # Bookmark table
    └── 20260723_feed_preferences.sql # Preference table
```

---

*This specification should be updated as new features are added. Each feature section should be marked ✅ COMPLETE, 🚧 IN PROGRESS, or 📋 PLANNED.*
