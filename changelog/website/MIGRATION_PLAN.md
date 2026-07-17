# Migration Plan: Pages Router → App Router

> **Target**: Next.js 16 App Router with React 19 Server Components
> **Current**: Next.js 16 Pages Router with React 19

---

## 1. Rationale

| Reason | Benefit |
|--------|---------|
| **Layouts** | Nested layouts eliminate manual layout wrappers |
| **Server Components** | Content rendering without client JS |
| **Streaming** | Progressive rendering for content-rich pages |
| **Loading states** | Built-in loading.tsx for each segment |
| **Error handling** | Built-in error.tsx boundaries |
| **SEO** | Better metadata API, improved SSR |
| **ISR** | Incremental Static Regeneration for content |

## 2. Migration Strategy

### Phase 1: App Router Setup (Parallel to Pages Router)

```
src/
├── app/                    # NEW: App Router
│   ├── layout.tsx          # Root layout (replaces _app.tsx)
│   ├── page.tsx            # Home page
│   └── loading.tsx         # Global loading
├── pages/                  # OLD: Pages Router (keep during migration)
│   └── ...
└── presentation/           # Shared components (no change)
    └── ...
```

### Phase 2: Route-by-Route Migration

Routes will be migrated one at a time, FROM:

```
src/pages/
├── index.tsx            → src/app/page.tsx
├── about/index.tsx      → src/app/about/page.tsx
├── contact/index.tsx    → src/app/contact/page.tsx
├── articles/index.tsx   → src/app/articles/page.tsx
├── articles/[slug].tsx  → src/app/articles/[slug]/page.tsx
├── roadmaps/index.tsx   → src/app/roadmaps/page.tsx
├── dashboard/index.tsx  → src/app/dashboard/page.tsx
├── profile/index.tsx    → src/app/profile/page.tsx
├── path/[...slug].tsx   → src/app/path/[...slug]/page.tsx
└── ...
```

### Phase 3: Remove Pages Router

Once ALL routes are migrated:
1. Delete `src/pages/` directory
2. Remove `_app.tsx` and `_document.tsx`
3. Update any import references

## 3. Key Changes

### _app.tsx → Root Layout

**Before** (`src/pages/_app.tsx`):
```tsx
export default function App({ Component, pageProps }: AppProps) {
  return (
    <>
      <Loading />
      <Navbar />
      <Component {...pageProps} />
    </>
  );
}
```

**After** (`src/app/layout.tsx`):
```tsx
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <Loading />
        <Navbar />
        {children}
        <Footer />
      </body>
    </html>
  );
}
```

### Route Handlers

**Before** (`src/pages/api/search.ts`):
```tsx
import type { NextApiRequest, NextApiResponse } from 'next';

export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  // ...
}
```

**After** (`src/app/api/search/route.ts`):
```tsx
import { NextRequest, NextResponse } from 'next/server';

export async function GET(request: NextRequest) {
  // ...
  return NextResponse.json({ data });
}
```

### Navigation

**Before**: `useRouter` from `next/router`
```tsx
import { useRouter } from 'next/router';
const router = useRouter();
router.push('/about');
```

**After**: `useRouter` from `next/navigation`
```tsx
import { useRouter } from 'next/navigation';
const router = useRouter();
router.push('/about');
```

### Dynamic Routes

**Before**: `src/pages/articles/[slug].tsx`
```tsx
import { useRouter } from 'next/router';
export default function Article() {
  const router = useRouter();
  const { slug } = router.query;
}
```

**After**: `src/app/articles/[slug]/page.tsx`
```tsx
export default async function Article({ params }: { params: { slug: string } }) {
  const { slug } = await params;
}
```

## 4. Component Migration Rules

1. **Server Components by Default**: All content-rendering components should be server components
2. **Client Components Only When Needed**: Only add `'use client'` for:
   - `useState` / `useEffect`
   - Event handlers (onClick, onSubmit)
   - Browser-only APIs
   - Context providers
3. **Data Fetching**: Use `async component` pattern for data fetching
4. **Metadata**: Use `export const metadata` or `generateMetadata()` for SEO

## 5. File Migration Checklist

Each route migration must:
- [ ] Create `src/app/[route]/page.tsx` (note: folder, not file)
- [ ] Create `src/app/[route]/layout.tsx` if nested layout needed
- [ ] Create `src/app/[route]/loading.tsx` for loading states
- [ ] Create `src/app/[route]/error.tsx` for error boundaries
- [ ] Update all import paths
- [ ] Replace `useRouter` from `next/router` with `next/navigation`
- [ ] Replace `NextApiRequest/Response` with `NextRequest/NextResponse`
- [ ] Test the route works
- [ ] Remove old `src/pages/[route]` file

## 6. Priority Order

1. `app/layout.tsx` (root layout)
2. `app/page.tsx` (home — replaces `src/pages/index.tsx`)
3. `app/about/` (simple static page)
4. `app/contact/` (simple static page)
5. `app/privacy/`, `app/terms/` (simple static pages)
6. `app/roadmaps/` (static with data)
7. `app/resources/` (static with data)
8. `app/articles/` (dynamic routes with [slug])
9. `app/path/` (catch-all routes)
10. `app/dashboard/` (authenticated)
11. `app/profile/` (authenticated)
12. `app/admin/` (authenticated)
13. `app/api/` (route handlers — full rewrite)
14. Delete `src/pages/`
