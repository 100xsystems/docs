# 100xSystems — Project Status & What's Next

> **Last updated:** July 2026
> **Author:** AI-assisted project audit

---

## Table of Contents

1. [What's Been Built](#1-whats-been-built)
2. [Architecture Overview](#2-architecture-overview)
3. [Website Audit](#3-website-audit)
4. [What's Next — Roadmap](#4-whats-next--roadmap)
5. [Known Issues](#5-known-issues)
6. [How to Contribute](#6-how-to-contribute)

---

## 1. What's Been Built

### 1.1 CLI (`@100xsystems/cli`)

The core learning tool. A terminal-based curriculum interface built with Ink + Pastel.

| Feature | Status | Details |
|---------|--------|---------|
| `100xsystems init` | ✅ Done | Scaffolds new projects. Interactive wizard + direct mode. Supports TypeScript and Java tracks. |
| `100xsystems validate` | ✅ Done | 3-level validation: L1 (structure), L2 (behavioral tests), L3 (spec checks). |
| `100xsystems submit` | ✅ Done | Creates GitHub PR with review package. |
| `100xsystems list` | ✅ Done | Lists available systems from registry. |
| `100xsystems doctor` | ✅ Done | Checks dev environment (Node, Git, Docker, Maven). |
| `100xsystems progress` | ✅ Done | Tracks per-lesson progress locally. |
| `100xsystems contribute` | ✅ Done | Scaffolds curriculum content (systems, tracks, lessons). |
| `100xsystems login` / `logout` | ✅ Done | GitHub OAuth via PKCE. |
| `100xsystems update` | ✅ Done | Checks for CLI updates. |

### 1.2 CLI Key Architectural Decisions

- **Test-runner executor** — Runs real behavioral tests (vitest for TS, JUnit for Java) in isolated temp directories
- **Registry-based system discovery** — Fetches `registry.json` from `100xsystems/registry` repo, shallow-clones system repos to `~/.cache/100xsystems/repos/`
- **3-level validation** — L1: project structure/file existence. L2: lesson behavioral tests. L3: spec-defined checks
- **Auto-detect `expected_passes`** — Parses test files for `it()`/`test()` calls (TS) or `@Test` annotations (Java)
- **npm dependency caching** — Warm `node_modules` cache at `~/.cache/100xsystems/test-node-modules/`
- **Temp dir registry** — Tracks temp dirs in `~/.cache/100xsystems/test-tmp-dirs.json`, cleans stale entries >1hr
- **Frontmatter validation** — Validates lesson frontmatter for required keys, known validator types, duplicate keys, malformed YAML

### 1.3 Test Suite Packages

| Package | Status | Description |
|---------|--------|-------------|
| `@100xsystems/test-suite-typescript` | ✅ Published (npm) | Shared vitest helpers — `fileExists`, `readFile`, `readJson`, `expectBuildSucceeds`, `withTempDir`, `withTempFile` |
| `@100xsystems/test-suite-java` | ✅ Published (GitHub Packages) | Shared JUnit 5 helpers — `BaseTest`, `FileHelper`, `BuildHelper`, `withTempDir`, `withTempFile` |

### 1.4 GitHub Organization (`github.com/100xsystems`)

| Repository | Status | Purpose |
|------------|--------|---------|
| `registry` | ✅ Live | Tiny JSON index of all available system repos |
| `cli` | ✅ Live | The 100xSystems CLI tool |
| `website` | ✅ (monorepo) | 100xsystems.dev website |
| `claude-code` | ✅ Live | System curriculum: Claude Code (TypeScript + Java tracks) |
| `system-template` | ✅ Live | Template for creating new system repos |
| `test-suite-typescript` | ✅ Live | Shared TypeScript test helpers (npm) |
| `test-suite-java` | ✅ Live | Shared Java test helpers (GitHub Packages) |
| `docs` | ✅ Live | Organization documentation |

### 1.5 Website (`100xsystems.dev`)

| Page | Status | Details |
|------|--------|---------|
| Homepage | ✅ Done | 10-section landing page with animations |
| Systems listing | ✅ Done | Lists all systems from curriculum |
| System detail | ✅ Done | Track/module/lesson hierarchy with CLI Quick Start |
| Lesson reader | ✅ Done | Markdown reading with outline, font controls, settings |
| CLI Docs | ✅ Done | Full CLI documentation from markdown files |
| Community | ⚠️ Mock data | Submission listings + leaderboard (falls back to mock) |
| Search | ✅ Done | Tag-based curated resource search |
| Knowledge Base | ✅ Done | Principles, Patterns, Tools, Technologies pages |
| Certificate verification | ✅ Done | Verifiable certificate pages |
| Languages | ⚠️ Coming Soon | Page exists, content coming |
| Templates / Specs | ⚠️ Routes exist | Pages need content |

### 1.6 Curriculum Structure

```
curriculum/
├── cli-docs/          # CLI documentation markdown files
├── knowledge-base/    # Principles, Patterns, Tools, Technologies
├── search/            # Tag-based resource JSON files
└── systems/           # System curriculum (one folder per system)
    └── [slug]/
        ├── index.md   # System metadata with track definitions
        └── track-{lang}/
            └── module-{n}/
                ├── lesson.md
                ├── quiz.md
                ├── challenge.md
                └── tests/
                    └── behavior.test.ts (or LessonTest.java)
```

---

## 2. Architecture Overview

### 2.1 System Discovery Flow

```
CLI startup
    │
    ├─► Check ~/.cache/100xsystems/repos/ for cached system repos
    │
    ├─► Fetch registry.json from github.com/100xsystems/registry
    │       └─ Contains: { slug, repo, version } for each system
    │
    └─► Shallow-clone missing repos into cache
            └─► System repo contains: curriculum/, tests/, docker/
```

### 2.2 Validation Flow

```
100xsystems validate
    │
    ├─► L1: Project Structure
    │       ├─  100xsystems.json valid?
    │       ├─  README.md exists?
    │       ├─  package.json / pom.xml exists?
    │       ├─  src/ directory exists?
    │       └─  .git initialized?
    │
    ├─► L1.5: Frontmatter Validation
    │       ├─  Required keys present? (title, description)
    │       ├─  Validator types recognized?
    │       ├─  Duplicate keys detected?
    │       └─  YAML well-formed?
    │
    ├─► L2: Lesson Validators (Executor-based)
    │       ├─  file-exists, file-contains, cli-command
    │       ├─  test-runner (vitest for TS, JUnit for Java)
    │       │       ├─  Copy source to temp dir
    │       │       ├─  Inject test-suite dependency
    │       │       ├─  Run tests (npx vitest / mvn test)
    │       │       └─  Parse results (JSON / XML reports)
    │       └─  docker, http, npm-test
    │
    └─► L3: Spec-Defined Checks
            └─  From SPECIFICATION.md in curriculum
```

### 2.3 Submission Flow

```
100xsystems submit
    │
    ├─► 1. Run validation (all levels)
    ├─► 2. Confirm with user
    ├─► 3. Authenticate via GitHub OAuth
    ├─► 4. Collect metadata
    ├─► 5. Build and package
    └─► 6. Create PR to 100xsystems/submissions
```

---

## 3. Website Audit

### 3.1 Pages & Routes

| Route | Status | Notes |
|-------|--------|-------|
| `/` | ✅ Live | 10-section homepage with hero, philosophy, cubix, etc. |
| `/systems` | ✅ Live | Lists all systems from curriculum |
| `/systems/[slug]` | ✅ Live | System detail with tracks, modules, lessons + CLI Quick Start |
| `/systems/[slug]/read/[...path]` | ✅ Live | Full reading experience — markdown, outline, settings |
| `/cli-docs` | ✅ Live | CLI documentation index with installation guide |
| `/cli-docs/[slug]` | ✅ Live | Individual CLI command docs |
| `/community` | ⚠️ Live | Submissions + leaderboard (mock data fallback) |
| `/search` | ✅ Live | Tag-based resource search |
| `/principles` | ✅ Live | Knowledge base — principles domain |
| `/patterns` | ✅ Live | Knowledge base — patterns domain |
| `/tools` | ✅ Live | Knowledge base — tools domain |
| `/technologies` | ✅ Live | Knowledge base — technologies domain |
| `/languages` | ⚠️ Coming Soon | Languages coming soon |
| `/verify/[certId]` | ✅ Live | Certificate verification page |
| `/systems/[slug]/templates` | ⚠️ Empty | Route exists, no content |
| `/systems/[slug]/specifications` | ⚠️ Empty | Route exists, no content |

### 3.2 Critical Gaps

| Gap | Severity | Why It Matters |
|-----|----------|----------------|
| **No dashboard/progress page** | 🔴 High | Users have no way to track what they've completed across systems |
| **No dark mode** | 🟡 Medium | 70% of devs prefer dark mode. Only light and sepia exist on reading pages |
| **Community data is mock** | 🟡 Medium | Falls back to mock because `GITHUB_TOKEN` env var isn't configured in prod |
| **No keyboard navigation** | 🟡 Medium | No `j`/`k` for prev/next lesson, no `/` for search |
| **No way to init from website** | 🟡 Medium | User reads about a system, has to manually open CLI — no "Get Started" flow |
| **No loading states** | 🟡 Medium | Community/search fetch data client-side with no skeleton states |
| **Languages page is dead** | 🟡 Medium | `getAllLanguages()` returns empty — no `languages/` dir in curriculum |

### 3.3 UX Issues

| Issue | Severity | Suggestion |
|-------|----------|------------|
| Homepage is heavy (10 sections + GSAP + motion) | 🟡 Medium | Lazy-load below-fold sections |
| No table of contents on mobile | 🟡 Medium | Lesson outline only shows on `xl:` screens |
| No error boundaries | 🟡 Medium | If a page throws, user sees white screen |
| Search page doesn't search curriculum | 🟡 Medium | Only searches tag-based resources, not lesson content |
| No reading time estimates | 🟢 Low | Lessons show `estimated_time` in frontmatter but website doesn't display it |
| Mobile lesson navigation cramped | 🟢 Low | MobileNav has settings/copy/fullscreen but no prev/next lesson |

### 3.4 Technical Concerns

| Concern | Why |
|---------|-----|
| GSAP ScrollSmoother may conflict with Next.js navigation | Wrapper gets destroyed/recreated on every route change |
| Large client bundles | `motion/react` + `gsap` + `highlight.js` on every page |
| Curriculum path hardcoded | `CURRICULUM_ROOT = path.join(process.cwd(), '..', 'curriculum')` only works in monorepo |

---

## 4. What's Next — Roadmap

### 🔴 Phase 1: Foundation (High Priority)

- [ ] **Progress/Dashboard page** — Read `~/.100x/progress.json` and render as a user dashboard
- [ ] **Dark mode** — Extend reading context theme system to full site
- [ ] **Get Started CTA** — "Build this system" button on every system page that copies install + init commands
- [ ] **Connect community to real data** — Configure `GITHUB_TOKEN` in deployment
- [ ] **Keyboard navigation** — `j`/`k` for prev/next lesson, `/` for search focus

### 🟡 Phase 2: Content & Polish (Medium Priority)

- [ ] **Language track comparison on system detail pages** — "Also available in Java / TypeScript"
- [ ] **Render lesson validation blocks as interactive checklists** on reading pages
- [ ] **Add loading/skeleton states** to community page and search page
- [ ] **Add error boundaries** to each route group
- [ ] **Add reading time estimates** to lesson cards on system detail page
- [ ] **Mobile lesson outline** — Collapsible outline on mobile reading pages

### 🟢 Phase 3: Growth (Lower Priority)

- [ ] **Blog page** — Footer links to it, doesn't exist yet
- [ ] **Custom 404 page**
- [ ] **RSS feed** for knowledge base articles
- [ ] **Print-friendly CSS** for lesson reading
- [ ] **Social sharing** for certificates
- [ ] **Search curriculum content** — Full-text search across all lessons
- [ ] **Recommended learning paths** — "Start with Claude Code, then move to Microservices"

### 🏗️ Infrastructure

- [ ] **Configure ISR** for content pages (revalidate on content update)
- [ ] **Dynamic imports** for GSAP and motion/react on pages that don't need them
- [ ] **Fix curriculum path resolution** — Support both monorepo and deployed (registry-cloned) layouts
- [ ] **Set up GITHUB_TOKEN** in Vercel/Netlify env vars for community page
- [ ] **Performance audit** — Lighthouse scores, bundle analysis, image optimization

---

## 5. Known Issues

### CLI
1. **Java track lessons don't have pom.xml or Java source files yet** — Only lesson markdowns exist. Students need to create their own Maven project.
2. **`runJUnit()` hasn't been tested end-to-end** — Java helpers compile inline but no real Maven project has been validated against them.
3. **`ensureMavenDependencies()` XML injection is string-based** — Could produce malformed XML if pom.xml has unusual formatting. Is reliable for standard pom.xml files.
4. **No Maven dependency caching** — Unlike npm caching for TypeScript, Java track installs dependencies fresh every time.

### Website
1. **Curriculum path is hardcoded to monorepo layout** — Only works when website is run from within the monorepo.
2. **Community page falls back to mock data** — Requires `GITHUB_TOKEN` env var for real data.
3. **No dark mode outside reading pages** — Theme toggle only affects the article reading experience.
4. **GSAP ScrollSmoother may conflict with Next.js App Router** — The smooth scrolling wrapper is created/destroyed on navigation, which can cause jank.

### Test Suite
1. **`@100xsystems/test-suite-java` needs `mvn deploy` to republish** — Changes to the package require a manual publish to GitHub Packages.
2. **Java helpers are compiled inline by the CLI** — The `copyTestSuiteJavaHelpers()` method copies source files into the test tree. This works but doesn't match the GitHub Packages dependency pattern.

---

## 6. How to Contribute

### Adding a New System
1. Run `100xsystems contribute init` to scaffold the system
2. Add tracks: `100xsystems contribute track`
3. Add lessons: `100xsystems contribute lesson`
4. Each lesson gets a `tests/` folder with boilerplate test file
5. Push to a new repo at `github.com/100xsystems/{slug}`
6. Update `registry.json` in the registry repo

### Adding a Lesson
1. Run `100xsystems contribute lesson` with system slug
2. Edit the generated `lesson.md` with content
3. Edit `tests/behavior.test.ts` (or `tests/LessonTest.java`) with behavioral tests

### Modifying the CLI
```bash
cd migration/100xsystems/cli
npm run build    # TypeScript compile + copy templates
npm link         # Use globally for testing
```

### Running the Website
```bash
cd website
npm run dev      # Local development on port 3000
npm run build    # Static site generation
```

---

*Generated: July 2026 — comprehensive audit of all 100xSystems components*
