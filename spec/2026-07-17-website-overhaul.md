# Spec: Website Overhaul — July 17, 2026

> **Status:** Proposed
> **Author:** AI-assisted spec
> **Target completion:** July 2026

---

## 1. Summary

The website needs to be overhauled to:
1. Sync with the CLI tool (same auth, same data)
2. Replace Supabase + Giscus with Clerk + Turso Cloud
3. Add the CodeCrafters/Exercism comparison to the homepage
4. Remove all old infrastructure (Supabase, Giscus/Disqus discussions)
5. Add a donation mechanism

---

## 2. Auth Migration: Supabase → Clerk

### Current State
- `@supabase/supabase-js` in package.json
- `src/infrastructure/supabase.ts` — singleton client
- `supabase.auth.signInWithOAuth()` for GitHub OAuth
- `supabase.auth.getUser()` / `getSession()` for session
- 7 database service files (profiles, progress, achievements, analytics, community, etc.)

### Target State
- `@clerk/nextjs` in package.json
- `middleware.ts` at root with `clerkMiddleware()`
- Root layout wrapped with `<ClerkProvider>`
- `auth()` helper in server components
- `<SignIn />`, `<SignUp />`, `<UserButton />` components in the UI
- **No database needed for auth** — Clerk handles user management

### Required Env Vars
```
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_...
CLERK_SECRET_KEY=sk_...
```

### Files to Create
- `middleware.ts` — Clerk middleware for route protection
- Update `app/layout.tsx` — wrap with ClerkProvider
- Update `app/layout-header-wrapper.tsx` — add UserButton

### Files to Delete
- `src/infrastructure/supabase.ts`
- `src/infrastructure/database/` (entire folder — profiles, progress, achievements, analytics, community services)
- `src/application/types/database.types.ts`
- All hooks that depend on supabase

---

## 3. Database Migration: Supabase → Turso Cloud

### Current State
- Supabase client with PostgreSQL
- 7 database service files using `supabase.from('table')` queries
- Tables: profiles, user_progress, achievements, analytics_events, community_members, etc.

### Target State
- Turso (libSQL) via `@libsql/client`
- Minimal schema — just user progress sync and donation records
- `drizzle-orm` for type-safe queries (optional — raw SQL is fine for simple needs)

### Required Env Vars
```
DATABASE_URL=libsql://[db-name]-[org-name].turso.io
DATABASE_AUTH_TOKEN=...
```

### Schema
```sql
-- Users table (progress tracking)
CREATE TABLE users (
  id TEXT PRIMARY KEY,          -- Clerk user ID
  github_username TEXT,
  display_name TEXT,
  created_at TEXT DEFAULT (datetime('now'))
);

-- Progress tracking per system
CREATE TABLE progress (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  user_id TEXT NOT NULL,
  system_slug TEXT NOT NULL,
  lesson_slug TEXT NOT NULL,
  completed_at TEXT DEFAULT (datetime('now')),
  UNIQUE(user_id, system_slug, lesson_slug)
);

-- Donations
CREATE TABLE donations (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  user_id TEXT NOT NULL,
  amount INTEGER NOT NULL,       -- cents
  message TEXT,
  created_at TEXT DEFAULT (datetime('now'))
);
```

### Files to Create
- `src/lib/db.ts` — Turso client setup
- `src/lib/schema.ts` — Drizzle schema (optional)

---

## 4. Homepage: Add Comparison Section

### What to Add
A new section between the philosophy and cubix sections showing a 3-column comparison table:

```
100xSystems vs CodeCrafters vs Exercism
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

┌──────────────┬──────────────┬──────────────┬──────────────┐
│              │ CodeCrafters │   Exercism   │ 100xSystems  │
├──────────────┼──────────────┼──────────────┼──────────────┤
│ Focus        │ Rebuild      │ Language     │ Build full   │
│              │ real tools   │ mastery      │ systems      │
├──────────────┼──────────────┼──────────────┼──────────────┤
│ Review       │ Auto-tests   │ Human        │ PR-based     │
│              │ only         │ mentors      │ peer review  │
├──────────────┼──────────────┼──────────────┼──────────────┤
│ Price        │ $40-60/month │ Free         │ 100% Free    │
│              │              │ (donations)  │ Forever      │
├──────────────┼──────────────┼──────────────┼──────────────┤
│ Language     │ 5 languages  │ 80+ lang.    │ Any language │
│ Support      │              │              │ (per system) │
├──────────────┼──────────────┼──────────────┼──────────────┤
│ Certificate  │ None         │ None         │ Verified     │
│              │              │              │ certificate  │
└──────────────┴──────────────┴──────────────┴──────────────┘
```

### Mobile
On mobile, the table becomes a horizontal scroll or a stacked card layout.

---

## 5. Remove Giscus / Disqus Discussion System

### Current State
- `.env` has 15+ Giscus-related env vars (NEXT_PUBLIC_GISCUS_REPO, category IDs, etc.)
- Discussion section on lesson reading pages
- Unused Disqus-related code

### Target State
- Remove all Giscus env vars from `.env`
- Remove discussion section from reading pages
- Remove all Giscus-related components and styles

### Files to Modify
- `.env` — remove all GISCUS_* vars
- Remove giscus discussion rendering from `SystemFileReadingClient.tsx` (if present)
- Remove related styles from `articles.styles.ts` (discussionSection, discussionTitle)

---

## 6. GitHub Env Vars Cleanup

### Current `.env`

| Variable | Keep? | Notes |
|----------|-------|-------|
| `GITHUB_BRANCH` | ❌ Remove | Old monorepo reference |
| `GITHUB_REPO` | ❌ Remove | Old monorepo reference |
| `GITHUB_REPO_PATH` | ❌ Remove | Old monorepo reference |
| `GITHUB_TOKEN` | ✅ Keep | Needed for community page |
| `GITHUB_USERNAME` | ❌ Remove | Old monorepo reference |
| All GISCUS_* | ❌ Remove | Replaced by nothing |
| All SUPABASE_* | ❌ Remove | Replaced by Clerk + Turso |
| `VERCEL_OIDC_TOKEN` | ✅ Keep | Deployment-related |

### Clean `.env` Target
```
GITHUB_TOKEN=ghp_...          # For community data
DATABASE_URL=libsql://...      # Turso
DATABASE_AUTH_TOKEN=...        # Turso
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_...
CLERK_SECRET_KEY=sk_...
```

---

## 7. Donation Mechanism

### Approach (Zero-Cost)
Since we're 100% free forever, use GitHub Sponsors:
1. Add a "Sponsor" button in the footer linking to `github.com/sponsors/100xsystems`
2. Add a donation section on the system detail pages
3. No Stripe/PayPal integration needed initially

### If GitHub Sponsors isn't available
Use Buy Me a Coffee or Ko-fi as a free alternative.

---

## 8. Implementation Order

1. **Docs restructure** ← We're here now ✅
2. **Write this spec** ← Doing this now ✅
3. **Get Clerk + Turso credentials** from user
4. **Remove old code** — Delete supabase.ts, database/, giscus, old env vars
5. **Add Clerk auth** — middleware.ts, ClerkProvider, UserButton, SignIn/SignUp
6. **Add Turso DB** — lib/db.ts, schema
7. **Add homepage comparison** — New section component
8. **Clean up env vars** — Remove old, add new
9. **Typecheck and test**
10. **Deploy**

---

## 9. Owner Checklist

- [ ] Provide Clerk publishable + secret keys
- [ ] Provide Turso database URL + auth token
- [ ] Confirm donation method (GitHub Sponsors vs Ko-fi)
- [ ] Approve spec before implementation begins
