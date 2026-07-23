# 100xSystems — Research Synthesis for Knowledge Operating System

> **Date:** July 23, 2026
> **Status:** Research Complete — Ready for Implementation
> **Constraint:** Zero-cost infrastructure, registry-level processing, no VPS

---

## Table of Contents

1. [The Four-Layer Architecture](#1-the-four-layer-architecture)
2. [Knowledge Graph (Layer 1)](#2-knowledge-graph-layer-1)
3. [GitHub Discovery (Layer 2)](#3-github-discovery-layer-2)
4. [Medium & Dev.to Integration (Layer 2)](#4-medium--devto-integration-layer-2)
5. [Reddit Integration (Layer 2)](#5-reddit-integration-layer-2)
6. [Static-Site Knowledge Graph Architecture](#6-static-site-knowledge-graph-architecture)
7. [Engineering Wikipedia — Navigation Design](#7-engineering-wikipedia--navigation-design)
8. [Historical Blog Import (Beyond RSS)](#8-historical-blog-import-beyond-rss)
9. [GitHub Actions Multi-Repo Pipelines](#9-github-actions-multi-repo-pipelines)
10. [Next.js Multi-Provider Architecture](#10-nextjs-multi-provider-architecture)
11. [Implementation Priority Matrix](#11-implementation-priority-matrix)

---

## 1. The Four-Layer Architecture

100xSystems is evolving from an RSS discovery platform into an **engineering knowledge operating system** with four layers:

```
┌──────────────────────────────────────────────────────────┐
│                    LAYER 4: SYSTEMS                       │
│                  (CloudyCode, Projects)                   │
│                                                          │
│  "Build a KV Store"  "Build a Chat App"  "Build a CDN"  │
├──────────────────────────────────────────────────────────┤
│                    LAYER 3: LEARNING                      │
│               (Courses, CLI, Test Suites)                 │
│                                                          │
│  track-typescript  track-java  track-spring-boot          │
├──────────────────────────────────────────────────────────┤
│                    LAYER 2: DISCOVERY                     │
│          (RSS, GitHub, Medium, Dev.to, Reddit)            │
│                                                          │
│  Engineering Blogs  GitHub Repos  Awesome Lists  Articles │
├──────────────────────────────────────────────────────────┤
│                    LAYER 1: KNOWLEDGE                     │
│                 (Entity Graph, Concepts)                  │
│                                                          │
│  ACID → CAP Theorem → MVCC → Paxos → Database Internals  │
└──────────────────────────────────────────────────────────┘
```

**Key insight:** Each layer is independent. Lower layers don't depend on upper layers. Upper layers reference entities from lower layers.

### Data Flow

```
Registry (runs locally / GitHub Actions)
  │
  ├── providers/rss/          → feeds/{id}.json          (✅ EXISTS)
  ├── providers/sitemap/      → feeds/{id}.json          (✅ PARTIAL)
  ├── providers/github/       → repos/{topic}.json       (🔜 NEXT)
  ├── providers/awesome/      → awesome/{name}.json      (🔜 PLANNED)
  ├── providers/devto/        → articles/{tag}.json      (🔜 FUTURE)
  └── providers/knowledge/    → entities/{id}.json       (🔜 PLANNED)
                              → manifest.json
  │
  ▼
Website (Next.js) reads static JSON files at build time
  │
  ├── /discover                → RSS + GitHub + Articles
  ├── /knowledge               → Knowledge Graph browser
  ├── /learn                   → Courses
  └── /build                   → Systems/Projects
```

---

## 2. Knowledge Graph (Layer 1)

### What We Learned

| Topic | Finding |
|-------|---------|
| **Best existing models** | Wikidata's triple-based approach (Subject-Predicate-Object). DBpedia extracts from Wikipedia infoboxes using OWL ontology. |
| **Lightweight storage** | JSON-LD (industry standard) or YAML-frontmatter in Markdown files. Version-controlled in git. |
| **Entity-first modeling** | Define entities as nodes with explicit `parents`, `children`, `related` fields referencing other entity IDs. |
| **Build-time graph** | SSG (Astro, Hugo, or Next.js) reads JSON files → generates static HTML for each entity. Client-side search via Lunr.js or FlexSearch. |
| **Existing ontologies** | ACM Computing Classification System (gold standard), IEEE SWEBOK, Basic Formal Ontology (BFO) for academic rigor. |

### Entity Schema Design

Every concept is a JSON entity with explicit relationships:

```json
{
  "id": "acid",
  "type": "concept",
  "title": "ACID",
  "aliases": ["ACID Transactions", "ACID Properties"],
  "summary": "ACID is a set of properties that guarantee database transactions are processed reliably.",
  "parents": ["database-transactions"],
  "children": ["atomicity", "consistency", "isolation", "durability"],
  "related": ["mvcc", "wal", "cap-theorem", "base"],
  "externalUrls": {
    "wikipedia": "https://en.wikipedia.org/wiki/ACID",
    "wikidata": "https://www.wikidata.org/wiki/Q215616"
  },
  "articles": [
    { "title": "A Primer on ACID Transactions", "url": "https://..." }
  ],
  "courses": [
    { "name": "Database Internals", "id": "database-internals" }
  ]
}
```

### Data Sources for Bootstrapping

1. **Wikidata SPARQL Queries** — Query the Wikidata Query Service for software engineering concepts and their relationships
2. **SWEBOK** — Software Engineering Body of Knowledge as a base taxonomy
3. **ACM CCS** — Computing Classification System for categories
4. **Manual Curation** — The best engineering concepts come from domain expertise

### Recommended Implementation Path

1. Define JSON schema → 2. Create entities directory → 3. Write first 50 entities (manually curated from SWEBOK + ACM CCS) → 4. SSG generates pages → 5. Deploy on GitHub Pages / Vercel

---

## 3. GitHub Discovery (Layer 2)

### What We Learned

| Topic | Finding |
|-------|---------|
| **API endpoint** | `GET /search/repositories?q=topic:distributed-systems` |
| **Rate limits (authenticated)** | 30 req/min for search, 5,000/hr for standard endpoints |
| **Rate limits (unauthenticated)** | 10 req/min for search — very restrictive |
| **Pagination** | `page` + `per_page` (max 100). Limited to first 1,000 results per query. |
| **Caching** | Store as JSON files with `last_updated` timestamp. Use `If-None-Match` with GitHub's `ETag`. |
| **Scraping** | ❌ Do NOT scrape GitHub HTML — violates ToS. Always use the API. |
| **Alternative** | [GH Archive](https://www.gharchive.org/) provides public datasets of GitHub events. |

### Awesome Lists — Special Category

**Why Awesome lists matter:**
- They are curated collections of links to the best resources in a domain
- Each Awesome list is essentially a human-curated knowledge graph
- Examples: `awesome-distributed-systems`, `awesome-database`, `awesome-python`, `awesome-microservices`
- The `README.md` of these repos contains categorized links with descriptions

**How to index Awesome lists:**

1. **Discover:** Search GitHub for repos with topic `awesome` or name starting with `awesome-`
2. **Fetch:** Get the README.md content via GitHub API (`GET /repos/{owner}/{repo}/readme`)
3. **Parse:** Extract categorized links from the markdown:
   - ```markdown
     ## Category Name
     - [Resource Title](https://...) - Description
     ```
4. **Store:** Each link becomes a registry entry with:
   - URL, title, description, category, source Awesome list, date indexed
5. **Deduplicate:** Same link may appear in multiple Awesome lists

**Ethical note:** We store only the link metadata (URL, title, description). Users click through to read.

### Registry-Level Processing

All GitHub data is processed at the **registry level** (local machine or GitHub Action), NOT at the website's API level:

```
Registry GitHub Action (daily or weekly):
  1. GitHub Search API → find repos by topic
  2. For Awesome lists: fetch README → parse links
  3. Store results as JSON files in registry/repos/
  4. Commit and push

Website:
  1. At build time, fetches JSON from registry
  2. Renders /discover/github pages
  3. No API rate limit concerns — all pre-cached
```

---

## 4. Medium & Dev.to Integration (Layer 2)

### What We Learned

| Source | Method | Rate Limits | Quality |
|--------|--------|-------------|---------|
| **Dev.to API** | `dev.to/api/articles?tag=...` | **Free, no auth**, paginated JSON | ⭐⭐⭐ High |
| **Medium tag RSS** | `medium.com/feed/tag/{tag}` | **Free**, latest 10-20 only | ⭐⭐ Medium |
| **Medium scraping** | Custom scraper | ❌ Violates ToS, Cloudflare blocks | ❌ Not recommended |
| **Hashnode API** | GraphQL endpoint | Requires API key | ⭐⭐⭐ High |

### Dev.to — Best First Integration

Dev.to has the best developer experience:
- No authentication required
- Structured JSON responses
- Pagination support
- Tags filter, top/week/month sort
- Rate limit: generous (no documented limit for read operations)

Example:
```
GET https://dev.to/api/articles?tag=distributed-systems&per_page=50&page=1
```

Response includes: title, url, description, tags, user, published_at, reading_time_minutes, page_views_count, positive_reactions_count

### Medium — Limited but Useful

Medium's tag-based RSS feeds work but have limitations:
- Only latest 10-20 articles per tag
- No pagination for historical data
- Paywalled articles show only truncated snippets

Still useful for: recent trending content in engineering tags

### Recommendation

1. **Start with Dev.to API** — most content, best API, no auth needed
2. **Add Medium tag RSS** — complementary, same processing pipeline
3. **Skip Medium scraping** — not worth the ToS risk and bot detection
4. **Store all as static JSON** in the registry, same pattern as RSS feeds

---

## 5. Reddit Integration (Layer 2)

### What We Learned

| Topic | Finding |
|-------|---------|
| **API access (2026)** | ⚠️ Significantly restricted. OAuth required. Developer approval difficult for small projects. |
| **Rate limits** | 100 queries/min per OAuth client (if approved) |
| **RSS feeds** | `reddit.com/r/programming/.rss` — historically worked but reliability has declined. |
| **Quality filtering** | Must be done programmatically: score thresholds, domain filtering, keyword matching. |
| **Alternative** | Hacker News API is free and easy: `https://hn.algolia.com/api/v1/search?query=...` |

### Verdict: Low Priority

Reddit API access in 2026 is too unreliable for a zero-cost project. **Hacker News** is a much better alternative with:
- Free API (no auth)
- Algolia-powered search
- Structured JSON responses
- High-quality engineering content

### Recommendation

Skip Reddit for now. Add Hacker News integration instead if more social/community content is needed.

---

## 6. Static-Site Knowledge Graph Architecture

### File Structure

```
registry/
├── knowledge/
│   ├── manifest.json            # Global index of all entities
│   └── entities/
│       ├── concepts/
│       │   ├── acid.json
│       │   ├── cap-theorem.json
│       │   ├── mvcc.json
│       │   ├── paxos.json
│       │   └── ... (50+ entities)
│       ├── technologies/
│       │   ├── postgresql.json
│       │   ├── kafka.json
│       │   └── redis.json
│       └── patterns/
│           ├── singleton.json
│           ├── observer.json
│           └── cqrs.json
├── repos/                        # GitHub repo cache
│   ├── distributed-systems.json
│   ├── databases.json
│   └── ai.json
└── articles/                     # Dev.to + Medium cache
    ├── distributed-systems.json
    └── databases.json
```

### Cross-Reference Resolution

At build time, the website:
1. Reads `manifest.json` → builds a Map of all entity IDs → metadata
2. For each entity, resolves `parents`, `children`, `related` array to actual entities
3. Pre-computes **backlinks**: for each entity, which other entities reference it
4. Passes this to the UI as static props

```typescript
// Build-time resolution
const backlinks = new Map<string, string[]>();
for (const entity of allEntities) {
  for (const related of entity.related ?? []) {
    const existing = backlinks.get(related) ?? [];
    existing.push(entity.id);
    backlinks.set(related, existing);
  }
}
```

### Navigation UI

Each entity page shows:
- Title + summary
- Parent concepts (breadcrumb: `Databases → Transactions → ACID`)
- Child concepts (expandable list)
- Related concepts (side panel or footer)
- Backlinks ("Referenced by: 12 other concepts")
- Articles, courses, repos that reference this entity

---

## 7. Engineering Wikipedia — Navigation Design

### Best Tools for a Wiki-Like Knowledge Base

| Tool | Pros | Cons | Best For |
|------|------|------|----------|
| **Next.js (in-house)** | Shares components with feed, ISR support, full control | More code to write | Our use case — already using Next.js |
| **MkDocs + Material** | Beautiful, search, versioning | Python, separate build | Standalone wiki |
| **Docusaurus** | React, versioning, Algolia search | Heavy for our needs | Large documentation |
| **Docsify** | No build step, markdown-driven | Client-side rendering only | Simple docs |

### Guided Exploration Pattern

To navigate from "ACID" → "Paxos" as a learning journey:

```
Concept: ACID
  ↓ (child)
  Isolation
    ↓ (related)
  MVCC
    ↓ (related)
  Distributed Transactions
    ↓ (related)
  Consensus Algorithms
    ↓ (child)
  Paxos
    ↓ (related)
  Raft
```

Each page has:
- **Breadcrumbs**: `Databases → Transactions → ACID → Isolation`
- **Up next**: Suggested next concept to learn
- **Related concepts**: Sidebar with connected topics
- **External resources**: Best articles, videos, courses for this concept

---

## 8. Historical Blog Import (Beyond RSS)

### What We Learned

| Method | Works For | Status |
|--------|-----------|--------|
| **Sitemap XML** | 18 of 36 feeds have sitemaps | ✅ Started — 1 feed tested (Dan Luu: 144 URLs, 16 new) |
| **Wayback Machine API** | All blogs have archive.org captures | 🔜 Future |
| **Pagination crawling** | Blogs without sitemaps (WordPress, Ghost) | 🔜 Future |
| **RSS-Bridge** | Sites without native RSS (generates RSS from HTML) | ⚠️ Fragile — bridges break |

### Already Implemented

The sitemap crawler (`registry/scripts/crawl-sitemap.ts`) works for 18 feeds:
- Cloudflare, CockroachDB, Confluent, ClickHouse, Dropbox, Tailscale, Pinecone, Julia Evans, Dan Luu, Rust, Stack Overflow, HashiCorp, Red Hat, Datadog, Figma, Grafana, Meta, Uber (removed — too broad)

### Next Steps for Historical Import

1. ✅ Run sitemap crawl for all 18 remaining feeds
2. 🔜 Test WayBack Machine CDX API for feeds without sitemaps
3. 🔜 Add pagination crawling for WordPress/Ghost blogs (Stripe, Discord, etc.)

---

## 9. GitHub Actions Multi-Repo Pipelines

### Communication Between Repos

| Method | Use Case |
|--------|----------|
| **`repository_dispatch`** | Async, event-driven: Repo A completes → triggers Repo B |
| **`gh workflow run`** | Sync, CLI-based: Wait for downstream workflow to complete |
| **`workflow_call`** | Reusable CI logic across repos |
| **Single data repo** | All data providers in one repo, easiest coordination |

### Recommendation: Single Data Repository

For simplicity, keep all data (feeds, repos, knowledge entities) in one repository:

```
Your local machine or GitHub Action
  │
  ├── npm run crawl:feeds        → fetches RSS, updates feeds/*.json
  ├── npm run crawl:sitemaps     → fetches sitemaps, updates feeds/*.json
  ├── npm run crawl:github       → fetches GitHub repos, updates repos/*.json
  ├── npm run crawl:awesome      → parses Awesome lists, updates awesome/*.json
  ├── npm run crawl:devto        → fetches Dev.to articles, updates articles/*.json
  └── npm run validate:all       → validates all JSON files
  │
  ▼
  git commit + git push
  │
  ▼
  Vercel rebuild (via webhook)
```

### Versioning

- Each commit is a versioned dataset
- Use git tags for stable releases
- Website can pin to a specific tag or use latest

---

## 10. Next.js Multi-Provider Architecture

### Feature-Driven Architecture

```
website/
├── app/
│   ├── discover/              # RSS + GitHub + Dev.to feeds
│   │   ├── page.tsx
│   │   ├── [tag]/             # Tag-filtered pages
│   │   └── bookmarks/         # User bookmarks
│   ├── knowledge/             # Knowledge graph browser
│   │   ├── page.tsx           # Browse all concepts
│   │   └── [concept]/         # Individual concept page
│   ├── learn/                 # Courses (existing)
│   │   └── ...
│   └── build/                 # Systems/Projects
│       └── ...
├── src/
│   ├── features/              # Feature modules (isolated)
│   │   ├── rss/               # ✅ Existing
│   │   ├── knowledge/         # 🔜 New
│   │   └── github/            # 🔜 New
│   ├── components/            # Shared UI primitives
│   │   ├── Button.tsx
│   │   ├── Card.tsx
│   │   └── Layout.tsx
│   └── lib/                   # Shared utilities
└── public/
    └── feed-cache.json        # Build-time artifact
```

### ISR Strategy

| Data Source | Update Frequency | ISR Strategy |
|-------------|-----------------|--------------|
| RSS feeds | Daily (GitHub Action) | `revalidate=86400` on feed pages |
| GitHub repos | Weekly | On-demand revalidation |
| Knowledge graph | Manual (curated) | Build-time only |
| Courses | Manual | Build-time only |

---

## 11. Implementation Priority Matrix

| Priority | Feature | Effort | Depends On | Cost |
|----------|---------|--------|-----------|------|
| **P0** | Document research (this doc) | 1h | None | $0 |
| **P1** | Knowledge Graph — schema + first 50 entities | 2-3 days | None | $0 |
| **P1** | Knowledge Graph — browse UI (`/knowledge/[concept]`) | 1 day | P1 entities | $0 |
| **P2** | GitHub repo crawler (by topic) | 1 day | GitHub token | $0 (auth token) |
| **P2** | GitHub Awesome list parser | 1 day | GitHub token | $0 (auth token) |
| **P2** | GitHub discovery UI (`/discover/github`) | 1 day | P2 crawler | $0 |
| **P3** | Dev.to topic API integration | 0.5 day | None | $0 |
| **P3** | Medium tag RSS integration | 0.5 day | None | $0 |
| **P3** | Knowledge Graph → RSS linking | 1 day | P1 + existing feed | $0 |
| **P4** | Sitemap crawl remaining 18 feeds | 2h | None | $0 |
| **P4** | Hacker News integration (alternative to Reddit) | 0.5 day | None | $0 |
| **Always** | Existing courses + CLI — keep untouched | 0 | None | — |

---

*End of Research Synthesis*
