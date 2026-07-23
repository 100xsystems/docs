# 100xSystems Engineering Discovery Feed System

> **Last updated:** July 23, 2026
> **Repos involved:** [`100xsystems/registry`](https://github.com/100xsystems/registry) + [`100xsystems/website`](https://github.com/100xsystems/website)

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Repository Layout](#2-repository-layout)
3. [How Scripts Work](#3-how-scripts-work)
4. [Running the Import Locally](#4-running-the-import-locally)
5. [GitHub Actions (Daily Cron)](#5-github-actions-daily-cron)
6. [Website Data Flow](#6-website-data-flow)
7. [Incremental Data Optimization (Delta System)](#7-incremental-data-optimization-delta-system)
8. [Adding a New Feed](#8-adding-a-new-feed)
9. [Troubleshooting](#9-troubleshooting)
10. [Architecture Diagrams](#10-architecture-diagrams)

---

## 1. Architecture Overview

The feed system is split across **two repositories** working together:

```
┌─────────────────────────────────────┐
│       REGISTRY REPOSITORY           │
│  (100xsystems/registry)             │
│                                     │
│  ┌─────────────────────────────┐    │
│  │  scripts/feed-registry.ts  │    │
│  │  → 51 feed definitions     │    │
│  └─────────────────────────────┘    │
│  ┌─────────────────────────────┐    │
│  │  scripts/update-feeds.ts   │    │
│  │  → Fetches RSS, writes     │    │
│  │    feeds/{id}.json         │    │
│  │    daily/delta.json        │    │
│  └─────────────────────────────┘    │
│  ┌─────────────────────────────┐    │
│  │  .github/workflows/        │    │
│  │  daily-feed-update.yml     │    │
│  │  → Runs daily at 00:00 UTC │    │
│  └─────────────────────────────┘    │
│                                     │
│  feeds/netflix-tech-blog.json       │
│  feeds/stripe-engineering.json      │
│  feeds/cloudflare-blog.json         │
│  ... (51 feed files)                │
│  daily/delta.json (latest 24h)      │
└───────────┬─────────────────────────┘
            │ GitHub raw + shallow clone
            ▼
┌─────────────────────────────────────┐
│       WEBSITE REPOSITORY            │
│  (100xsystems/website)              │
│                                     │
│  scripts/fetch-feed-cache.mjs       │
│  → Runs during next build (prebuild)│
│  → Clones registry, reads all feeds │
│  → Generates public/feed-cache.json │
│                                     │
│  src/feed/feed.cache.ts             │
│  → Loads cache at runtime           │
│  → Tiered: /tmp/ > public/ > raw    │
│  → Delta merge for ISR updates      │
│                                     │
│  app/api/feed/route.ts              │
│  → Reads from cache                 │
│  → Filter, sort, paginate           │
│  → Cache-Control: 24h edge cache    │
│                                     │
│  public/feed-cache.json             │
│  → Build-time artifact (deployed)   │
│  /tmp/feed-cache.json               │
│  → ISR revalidation output          │
└─────────────────────────────────────┘
```

### Key Design Principles

| Principle | Implementation |
|-----------|---------------|
| **No content storage** | We index ONLY metadata (title, URL, summary, author, date). Never article body. |
| **Zero backend** | No database, no VPS. RSS feeds are indexed by GitHub Actions, cached in the website. |
| **Incremental by default** | Only new items are fetched on each cycle via the delta system. |
| **Resilient** | 3-tier cache fallback. Edge cache serves stale on error for up to 30 days. |

---

## 2. Repository Layout

### Registry Repository (`100xsystems/registry`)

```
registry/
├── .github/workflows/
│   └── daily-feed-update.yml    # Cron job — runs daily at 00:00 UTC
├── scripts/
│   ├── feed-registry.ts         # Feed definitions (51 sources)
│   └── update-feeds.ts          # Main script: fetch RSS → write JSON
├── feeds/                       # Generated: one JSON file per feed
│   ├── netflix-tech-blog.json
│   ├── stripe-engineering.json
│   ├── cloudflare-blog.json
│   └── ... (51 files)
├── daily/                       # Generated: incremental delta files
│   └── delta.json               # Today's new items (tiny file)
├── package.json
└── tsconfig.json
```

### Website Repository (`100xsystems/website`)

```
website/
├── app/
│   ├── feed/
│   │   ├── page.tsx             # Feed landing page
│   │   ├── [tag]/
│   │   │   └── page.tsx         # Tag-specific pages (60+ SSG)
│   │   └── bookmarks/
│   │       └── page.tsx         # User bookmarks
│   └── api/feed/
│       └── route.ts             # JSON API endpoint
├── src/feed/
│   ├── feed.constants.ts        # Canonical feed registry (mirrors registry)
│   ├── feed.types.ts            # TypeScript interfaces
│   ├── feed.cache.ts            # Cache loading + delta merging
│   ├── feed.api.ts              # Client-side API calls
│   ├── feed.utils.ts            # Ranking, sorting, helpers
│   ├── FeedPage.tsx             # Main feed client component
│   ├── FeedHeader.tsx           # Header with filters + tags
│   ├── ArticleCard.tsx          # Single article component
│   ├── useBookmarks.ts          # Bookmark hook (localStorage + Turso)
│   └── feed.types.ts            # All types
├── scripts/
│   └── fetch-feed-cache.mjs     # Build-time cache generator
├── public/
│   └── feed-cache.json          # Generated: combined cache (deployed)
└── docs/spec/
    └── 2026-07-23-engineering-discovery-platform.md
```

---

## 3. How Scripts Work

### 3.1 `registry/scripts/update-feeds.ts`

**Purpose:** Fetches RSS feed items and writes them to JSON files.

**Two modes:**

| Mode | Flag | Items/feed | When to use |
|------|------|-----------|-------------|
| Incremental | (default) | 50 | Daily cron updates |
| Historical | `--historical --limit=500` | Up to 500 | First-time setup |

**What it does for each feed:**

1. Reads the existing `feeds/{id}.json` if it exists
2. Builds a `Set` of existing GUIDs (for deduplication)
3. Fetches the RSS feed via `rss-parser`
4. For each new item:
   - Extracts GUID (falls back to link, then title hash)
   - Converts relative URLs to absolute
   - Truncates summary to 300 chars
   - Appends to the item list
5. Caps total items at 10,000 per feed (to keep files manageable)
6. Writes atomically: temp file → rename

**On non-historical runs, also writes `daily/delta.json`:**
- Contains only the items added in this run
- Always small (< 50 new items across all feeds)
- Used by the website for incremental ISR updates

### 3.2 `registry/scripts/feed-registry.ts`

**Purpose:** Central list of all 51 feed definitions.

Each entry contains:
```typescript
{
  id: 'netflix-tech-blog',       // Unique slug
  name: 'Netflix Tech Blog',      // Display name
  rssUrl: 'https://...',          // RSS feed URL
  siteUrl: 'https://...',         // Website URL
  tags: ['distributed-systems'],  // Categories
}
```

**Note:** This file must mirror `website/src/feed/feed.constants.ts`. Update both when adding/removing feeds.

### 3.3 `website/scripts/fetch-feed-cache.mjs`

**Purpose:** Build-time script that generates `public/feed-cache.json`.

**Runs during:** `npm run prebuild` (which runs before `next build`)

**Two strategies (tried in order):**

| Strategy | When it works | Speed |
|----------|--------------|-------|
| **A: Copy from local** | Development (registry is sibling directory) | Instant |
| **B: Shallow clone** | CI/Vercel (no local registry) | ~10-30s |

**What it generates:**
```json
{
  "version": 2,
  "updatedAt": "2026-07-23T10:00:00Z",
  "feedCount": 36,
  "totalItems": 2019,
  "feeds": {
    "netflix-tech-blog": {
      "feedId": "netflix-tech-blog",
      "feedName": "Netflix Tech Blog",
      "items": [ ... ],
      "totalIndexed": 98
    }
  }
}
```

### 3.4 `website/src/feed/feed.cache.ts`

**Purpose:** Loads the feed cache at runtime with a 3-tier fallback hierarchy.

**Tier priority:**

```
1. /tmp/feed-cache.json        ← Written by ISR revalidation
   ↓ (not found)
2. public/feed-cache.json       ← Build-time artifact (deployed)
   ↓ (stale > 24h)
3. GitHub raw                   ← Fetch + write to /tmp/
```

**Incremental delta logic:**
- First tries fetching `daily/delta.json` from GitHub raw (tiny file)
- If found and newer than existing cache, merges new items into `/tmp/feed-cache.json`
- Only falls back to fetching all 51 feeds if no existing cache exists (cold start)

---

## 4. Running the Import Locally

### 4.1 Prerequisites

```bash
# Need Node.js 20+ and npm
node --version  # v20+
npm --version

# Clone both repos (they should be siblings)
git clone https://github.com/100xsystems/registry.git
git clone https://github.com/100xsystems/website.git
cd registry

# Install dependencies
npm install
```

### 4.2 First-Time Historical Import

```bash
cd registry

# Import up to 500 items per feed for ALL 51 feeds
npm run historical-import

# Or import for a single feed to test
npx tsx scripts/update-feeds.ts --historical --limit=200 --feed=netflix-tech-blog
```

This creates `feeds/{id}.json` files for each feed.

**Expected output:**
```
🔍 100xSystems Feed Updater
   Mode: HISTORICAL IMPORT
   Max items per feed: 500
   Target feeds: ALL (51)

  [netflix-tech-blog] Netflix Tech Blog... ✓ +42 new items (total: 42)
  [stripe-engineering] Stripe Engineering... ✓ +18 new items (total: 18)
  ...
```

### 4.3 Daily Incremental Update

```bash
cd registry

# Default: fetch latest 50 items per feed
npm run update-feeds

# Or with explicit options
npx tsx scripts/update-feeds.ts --limit=50
```

This fetches only the **latest items** and appends them to existing JSON files. Also writes `daily/delta.json` with only the new items.

### 4.4 Regenerate Website Cache Locally

```bash
cd website

# This copies feeds from ../registry/feeds/ and generates public/feed-cache.json
npm run fetch-feed-cache

# Or run it manually
node scripts/fetch-feed-cache.mjs
```

### 4.5 Full Local Workflow

```bash
# Step 1: Import feeds into registry
cd ../registry
npm run historical-import

# Step 2: Generate website cache
cd ../website
npm run fetch-feed-cache

# Step 3: Start dev server
npm run dev

# Step 4: Test the API
curl http://localhost:3000/api/feed?limit=10
```

---

## 5. GitHub Actions (Daily Cron)

### 5.1 Workflow File

Located at `registry/.github/workflows/daily-feed-update.yml`.

**Triggers:**
- **Scheduled:** Every day at 00:00 UTC (`cron: '0 0 * * *'`)
- **Manual:** Via GitHub UI (`workflow_dispatch`) with options:
  - `historical`: Boolean — run full historical import
  - `limit`: Number — max items per feed
  - `feed`: String — single feed to update (empty = all)

**Steps:**
1. Checkout repo (full history for diff)
2. Setup Node.js 22
3. Install dependencies (`npm ci`)
4. Run `update-feeds.ts` with appropriate flags
5. Check git diff for changes
6. If changes detected, commit and push

**Common scenario (99% of runs):**
- Runs the incremental update (50 items/feed)
- Most feeds have 0 new items → no commit
- A few feeds (Netflix, Cloudflare, etc.) have 1-3 new items → commit with `"chore(feeds): daily update — 2026-07-23"`
- The commit also includes the updated `daily/delta.json`

### 5.2 Why GitHub Actions is Free for This

| Resource | Usage | Free tier limit |
|----------|-------|-----------------|
| Run minutes | ~2 min/day = ~60 min/month | 2,000 min/month (public repos) |
| Storage | ~5MB per feed JSON | 500 MB (public repos) |
| Bandwidth | ~100KB per daily update | 1 GB/month (public repos) |

Since the registry repo is **public**, GitHub Actions is **completely free** with generous limits.

---

## 6. Website Data Flow

### 6.1 Build Time

```
next build
  │
  ▼
npm run prebuild
  │
  ├── scripts/sync-curriculum.mjs
  ├── scripts/copy-content-images.mjs
  │
  └── scripts/fetch-feed-cache.mjs
        │
        ├── Try: copy from ../registry/feeds/
        │   (fast, used in development)
        │
        └── Fallback: git clone registry (shallow)
            (used in CI/Vercel)
              │
              ▼
        public/feed-cache.json
        ← Deployed with static assets
```

### 6.2 Runtime (API Request)

```
User visits /feed
  │
  ▼
Browser calls GET /api/feed?feeds=netflix,stripe&limit=50
  │
  ▼
Vercel Edge Cache
  │
  ├── HIT (Cache-Control: s-maxage=86400)
  │   → Return cached response (~5ms)
  │
  └── MISS (stale > 24h OR first request after deploy)
      │
      ▼
    Serverless Function
      │
      ├── loadFeedCache()
      │   ├── Try /tmp/feed-cache.json (ISR revalidation)
      │   ├── Try public/feed-cache.json (build artifact)
      │   └── Fetch from GitHub raw + write to /tmp/
      │
      ├── Filter by requested feeds
      ├── Sort by publishedAt
      ├── Apply cursor pagination
      │
      └── Return JSON + Cache-Control headers
```

### 6.3 ISR Revalidation (24h Cycle)

```
Cache-Control: s-maxage=86400 expires
  │
  ▼
Next request triggers ISR revalidation
  │
  ▼
loadFeedCache() detects stale cache (>24h)
  │
  ├── ⚡ Fetch daily/delta.json from GitHub raw (TINY — < 1KB)
  │   │
  │   ├── Delta found + newer than cache
  │   │   → Merge new items into /tmp/feed-cache.json
  │   │   → Update updatedAt timestamp
  │   │
  │   └── No delta or delta is old
  │       → Fall back to fetching all 51 feeds
  │
  └── Return cache (serves immediately, background refresh)
```

### 6.4 Cache Hierarchy Summary

| Level | Location | Speed | When used |
|-------|----------|-------|-----------|
| **Edge** | Vercel CDN | ~5ms | Every request (Cache-Control) |
| **ISR** | `/tmp/feed-cache.json` | ~1ms | Between revalidations |
| **Build** | `public/feed-cache.json` | ~1ms | Cold start (new deploy) |
| **Registry RAW** | GitHub CDN | ~200ms | Stale cache, fetch delta |
| **Registry clone** | Full git clone | ~10s | First build (CI only) |

---

## 7. Incremental Data Optimization (Delta System)

### 7.1 The Problem

When the website's ISR triggers (every 24h), the naive approach would fetch ALL 51 feed JSON files from GitHub raw — each possibly containing thousands of historical items. This means:

- **51 HTTP requests** (one per feed)
- **MBs of data** (some feeds have 500+ items)
- **Slow revalidation** (parallel fetch of 51 files → network-bound)
- **Wasted bandwidth** (most feeds haven't changed since yesterday)

### 7.2 The Solution: Delta.json

The registry repo maintains a `daily/delta.json` file that contains ONLY the items added in the most recent update.

**Format:**
```json
{
  "date": "2026-07-23",
  "generatedAt": "2026-07-23T00:05:00Z",
  "commitSha": "abc123def456",
  "items": {
    "netflix-tech-blog": [
      {
        "guid": "netflixtechblog.com/scaling-cassandra",
        "title": "Scaling Cassandra at Netflix",
        "link": "https://netflixtechblog.com/scaling-cassandra",
        "summary": "How Netflix scaled their Cassandra infrastructure...",
        "author": "Netflix Engineering",
        "publishedAt": "2026-07-22T10:00:00.000Z"
      }
    ],
    "cloudflare-blog": [
      {
        "guid": "blog.cloudflare.com/ddos-july-2026",
        "title": "DDoS trends in Q3 2026",
        "link": "https://blog.cloudflare.com/ddos-july-2026",
        "summary": "Analysis of DDoS attack patterns...",
        "author": "Cloudflare Team",
        "publishedAt": "2026-07-22T14:30:00.000Z"
      }
    ]
  },
  "totalNewItems": 3,
  "feedCount": 51
}
```

**Size:** Typically < 1KB (3-5 new items across all 51 feeds).

### 7.3 How It Works End-to-End

```
00:00 UTC — GitHub Action triggers
  │
  ▼
update-feeds.ts runs (incremental, limit=50)
  │
  ├── Fetches RSS for each feed
  ├── Finds new items (deduplicates by GUID)
  ├── Appends them to feeds/{id}.json
  ├── Writes daily/delta.json with ONLY new items
  │
  ▼
Git commit: "chore(feeds): daily update — 2026-07-23"
  │
  ▼
Registry repo updated on GitHub
  │
  ▼
───── Several hours pass ─────
  │
  ▼
User visits /feed (cache expired)
  │
  ▼
ISR revalidation triggers
  │
  ▼
loadFeedCache() checks cache staleness
  │
  ├── Cache is >24h old
  │
  ▼
Fetch daily/delta.json from GitHub raw
  │
  ├── HTTP 200: delta found
  │
  ▼
Compare delta.generatedAt vs cache.updatedAt
  │
  ├── Delta is newer → merge
  │
  ▼
Merge delta.items into /tmp/feed-cache.json
  │
  ├── For each feed in delta:
  │   ├── Read existing feed data from cache
  │   ├── Prepend new items (newest first)
  │   ├── Remove duplicates (by GUID)
  │   └── Cap at 10,000 items
  │
  ▼
Write updated cache to /tmp/feed-cache.json
  │
  ▼
Update cache.updatedAt = delta.generatedAt
  │
  ▼
API serves fresh data (with new items)
```

### 7.4 Cold Start Handling

On a **cold start** (no `/tmp/` cache and no `public/feed-cache.json` yet):

```
No existing cache found
  │
  ▼
Try fetching daily/delta.json
  │
  └── Cannot use delta alone — no base to merge into
  │
  ▼
Fall back to fetching ALL feed files from GitHub raw
  │
  ├── 51 parallel HTTP requests
  ├── Each: raw.githubusercontent.com/.../feeds/{id}.json
  ├── Total: ~2-5 MB (all feeds)
  │
  ▼
Merge into /tmp/feed-cache.json
  │
  ▼
Write to /tmp/
  │
  ▼
Future requests use /tmp/ + delta
```

Cold starts are rare — they happen:
- On the very first deploy
- When Vercel's `/tmp/` ephemeral storage is wiped (infrequent)
- After a long period of inactivity

### 7.5 Fallback Chain Summary

```typescript
async function loadFeedCache(): Promise<FeedCache | null> {
  // 1. Try /tmp/ (ISR revalidation)
  //    If valid → return immediately
  const tmpCache = readCacheFromDisk(TMP_CACHE_PATH);
  if (tmpCache) return tmpCache;

  // 2. Try build cache
  const buildCache = readCacheFromDisk(BUILD_CACHE_PATH);
  if (buildCache) {
    // 2a. Not stale → return build cache
    if (!isCacheStale(BUILD_CACHE_PATH)) return buildCache;

    // 2b. Stale → try delta merge
    const delta = await fetchDeltaFromRegistry();
    if (delta && delta.generatedAt > buildCache.updatedAt) {
      const merged = mergeDelta(buildCache, delta);
      writeCacheToTmp(merged);
      return merged;
    }

    // 2c. Delta unavailable → full re-fetch
    const fresh = await fetchAllFeedsFromRegistry();
    return fresh ?? buildCache;
  }

  // 3. Nothing local → try delta first (new deploy, no data yet)
  const delta = await fetchDeltaFromRegistry();
  if (delta) {
    // Fetch all feeds as base, then apply delta
    const full = await fetchAllFeedsFromRegistry();
    if (full) {
      const merged = mergeDelta(full, delta);
      writeCacheToTmp(merged);
      return merged;
    }
  }

  // 4. Last resort: fetch all feeds
  return fetchAllFeedsFromRegistry();
}
```

### 7.6 What If The Website Misses A Few Days?

If the ISR doesn't trigger for 2-3 days (low traffic):

1. The delta.json from the latest day is fetched
2. But the cache is 3 days stale — the delta only has 1 day of new items
3. Missing 2 days of items is **acceptable** because:
   - The edge cache is still valid (Cache-Control: 24h)
   - Fresh data is fetched on the next request
   - Historical items don't change — only new items are lost
4. If the cache is >72h stale, the system falls back to the full fetch

---

## 8. Adding a New Feed

### 8.1 Find the RSS URL

Most engineering blogs have RSS at predictable URLs:
- `blog.example.com/feed`
- `blog.example.com/rss`
- `blog.example.com/feed.xml`

**Common patterns:**

| Platform | RSS URL |
|----------|---------|
| Medium | `medium.com/feed/@user` or `medium.com/feed/publication` |
| Ghost | `blog.com/rss` |
| WordPress | `blog.com/feed` |
| Hashnode | `blog.com/rss.xml` |
| Substack | `substack.com/feed` |

### 8.2 Add to Registry

**Step 1:** Edit `registry/scripts/feed-registry.ts` and add:
```typescript
{
  id: 'my-new-blog',
  name: 'My New Blog',
  rssUrl: 'https://mynewblog.com/feed',
  siteUrl: 'https://mynewblog.com',
  tags: ['relevant-tag-1', 'relevant-tag-2'],
}
```

**Step 2:** Edit `website/src/feed/feed.constants.ts` and add the SAME entry:
```typescript
{
  id: 'my-new-blog',
  name: 'My New Blog',
  rssUrl: 'https://mynewblog.com/feed',
  siteUrl: 'https://mynewblog.com',
  tags: ['relevant-tag-1', 'relevant-tag-2'],
  description: 'A short description of what this blog covers.',
  addedBy: 'curator',
  language: 'en',
}
```

### 8.3 Run Historical Import

```bash
cd registry

# Import just the new feed
npx tsx scripts/update-feeds.ts --historical --limit=200 --feed=my-new-blog
```

### 8.4 Regenerate Website Cache

```bash
cd website

# Regenerate public/feed-cache.json
node scripts/fetch-feed-cache.mjs
```

### 8.5 Rebuild and Deploy

```bash
cd website
npm run build
# Deploy to Vercel
```

### 8.6 Commit Registry Changes

```bash
cd registry
git add .
git commit -m "feat(feeds): add My New Blog"
git push
```

The GitHub Action will pick up the new feed on the next daily run.

---

## 9. Troubleshooting

### 9.1 "No items in feed" for a known-working URL

**Cause:** The RSS URL may have changed or the feed may be empty.

**Fix:**
1. Verify the URL works: `curl -s <rssUrl> | head -50`
2. Check if the site changed platforms (e.g., Medium → Ghost)
3. Update the RSS URL in both `feed-registry.ts` and `feed.constants.ts`

### 9.2 "XML parse error" for a feed URL

**Cause:** The URL doesn't point to valid RSS/Atom XML.

**Fix:**
1. Visit the site's blog page
2. Look for the RSS/feed link in the HTML `<head>` or footer
3. Try common patterns: `/feed`, `/rss`, `/feed.xml`, `/blog/feed`
4. Check if the site uses a specific platform that has a known RSS pattern

### 9.3 API returns empty cache

**Cause:** `public/feed-cache.json` doesn't exist or is empty.

**Fix:**
```bash
cd website
node scripts/fetch-feed-cache.mjs
```

Make sure the registry repo is accessible as a sibling directory or set `REGISTRY_REPO` env var.

### 9.4 Build fails because fetch-feed-cache hangs

**Cause:** GitHub clone taking too long or rate limiting.

**Fix:** Increase timeout or set up a local registry clone:
```bash
# Clone registry as sibling to website
cd ..
git clone https://github.com/100xsystems/registry.git
cd website
npm run fetch-feed-cache
```

### 9.5 GitHub Action fails with "Process completed with exit code 1"

**Cause:** One or more feed URLs returned errors.

**Fix:** 
1. Check the Action logs for which feeds failed
2. Verify those RSS URLs are still valid
3. Run locally to debug: `npx tsx scripts/update-feeds.ts --feed=broken-feed`

---

## 10. Architecture Diagrams

### 10.1 High-Level Data Flow

```
┌──────────┐     ┌──────────────┐     ┌──────────────┐
│  RSS     │────▶│  Registry    │────▶│  Website     │
│  Sources │     │  GitHub Repo │     │  (Next.js)   │
│  (51)    │     │              │     │              │
│          │     │  feeds/*.json│     │  /api/feed   │
│ Netflix  │     │  daily/      │     │  /feed       │
│ Stripe   │     │  delta.json  │     │  /feed/[tag] │
│ Cloudfl. │     │              │     │  /feed/bkmrk │
│ Discord  │     │  GitHub      │     │              │
│ ...      │     │  Action      │     │  Vercel      │
└──────────┘     │  (daily cron)│     │  (Edge CDN)  │
                 └──────────────┘     └──────────────┘
```

### 10.2 Cache Decision Tree

```
Request to /api/feed
  │
  ├── Edge Cache HIT?
  │   └── Yes → Return cached response (~5ms)
  │
  └── Edge Cache MISS?
      │
      ├── /tmp/feed-cache.json exists?
      │   ├── Yes → Return cache (~1ms)
      │   │
      │   └── No → public/feed-cache.json exists?
      │       │
      │       ├── Yes → Is it fresh (< 24h)?
      │       │   ├── Yes → Return cache (~1ms)
      │       │   │
      │       │   └── No → Fetch delta.json from GitHub
      │       │       │
      │       │       ├── Delta found and newer?
      │       │       │   ├── Yes → Merge into /tmp/ → Return
      │       │       │   └── No → Fetch all feeds → Merge into /tmp/ → Return
      │       │       │
      │       │       └── Delta failed → Fetch all feeds → Return (or stale)
      │       │
      │       └── No (cold start) → Fetch delta.json → Fetch all feeds → Write /tmp/ → Return
      │
      └── Set Cache-Control: public, s-maxage=86400
```

### 10.3 Cost Analysis

| Component | Cost | Details |
|-----------|------|---------|
| GitHub Actions | **$0** | Free for public repos (2,000 min/month) |
| GitHub Raw CDN | **$0** | Unlimited bandwidth for public repos |
| Vercel Hobby | **$0** | 100 GB bandwidth/month, 6,000 build min/month |
| Vercel Edge | **$0** | Included in Hobby (Edge caching built-in) |
| Registry Storage | **$0** | Public repo, ~5 MB total |
| **Total** | **$0/month** | |

### 10.4 Request Flow Example

```
User: GET /api/feed?limit=10
  │
  ├── Vercel Edge: Cache MISS (first request or expired)
  │
  ├── Serverless Function:
  │   ├── loadFeedCache()
  │   │   ├── /tmp/feed-cache.json: NOT FOUND (cold start)
  │   │   ├── public/feed-cache.json: FOUND (but stale > 24h)
  │   │   └── Fetching daily/delta.json from GitHub raw...
  │   │       └── 304 Not Modified (no new items today) → keep current cache
  │   │
  │   ├── Filter: all feeds (no ?feeds= param)
  │   ├── Sort: newest first
  │   ├── Paginate: first 10 articles
  │   │
  │   └── Response: JSON with 10 articles
  │       └── Cache-Control: public, s-maxage=86400, stale-while-revalidate=2592000
  │
  └── Vercel Edge: Caches response for 24h
      └── Next request → Edge HIT → ~5ms response
```

---

*End of documentation.*
