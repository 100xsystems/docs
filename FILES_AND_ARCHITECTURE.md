# 100xSystems — Complete Codebase Audit & Architecture

> **Date:** July 15, 2026
> **Scope:** Full audit of `curriculum/`, `cli/`, `website/`, `submissions/`, `.github/`

---

## 1. REPOSITORY ARCHITECTURE

```
100xsystems/                          # Monorepo root
├── curriculum/                       # 📚 Source of truth — all content
│   ├── systems/                      #    System-specific content
│   ├── knowledge-base/               #    Cross-cutting reference docs
│   ├── search/                       #    Curated resource JSONs
│   └── languages/                    #    (empty — future)
├── cli/                              # 🖥️ CLI tool — interactive learning
│   ├── src/
│   │   ├── commands/                 #    17 Pastel commands
│   │   ├── actions/                  #    5 action modules
│   │   ├── executors/                #    10 validator plugins (NEW)
│   │   ├── readers/                  #    5 curriculum readers
│   │   ├── ui/                       #    10+ Ink UI components
│   │   ├── auth/                     #    PKCE + Device Flow auth
│   │   └── scaffold/                 #    Project scaffolding
│   └── templates/                    #    Language templates
├── website/                          # 🌐 Next.js 15 SSG website
│   ├── app/                          #    App Router pages
│   │   ├── systems/                  #    System listing + detail + read
│   │   ├── principles/ etc.          #    Knowledge domain pages
│   │   ├── search/                   #    Search page
│   │   └── api/                      #    GitHub auth API route
│   ├── src/
│   │   ├── lib/                      #    mdx.ts — curriculum reader
│   │   ├── components/               #    Reading toolbar, MDX widgets
│   │   ├── presentation/             #    Design system + features
│   │   └── infrastructure/           #    Supabase client (used by service imports)
│   └── public/                       #    Static assets
├── submissions/                      # 📥 Review packages (local only — no PR flow)
│   ├── .github/workflows/            #    Certificate gen workflow (monorepo context — needs migration to separate submissions repo for full PR flow to work)
│   └── .gitkeep
├── docs/                             # 📝 Architecture + strategy docs
└── .github/workflows/                # CI/CD (publish)
```

---

## 2. CURRICULUM — CURRENT STATE

### 2.1 Systems — Active Content

| System | Tracks | Modules | Lessons | Status |
|--------|--------|---------|---------|--------|
| **Claude Code** | 2 (TypeScript, Java) | 4 TS, 0 Java | 6 TS + quiz + challenge | 🟡 Partial |
| **Microservices** | 2 (Spring Boot, NestJS) | 3 SB, 0 NestJS | 3 SB | 🟡 Partial |

### 2.2 Systems — Lesson Inventory

**Claude Code — TypeScript Track (`track-typescript/`):**
| Module | Lessons | Status |
|--------|---------|--------|
| Module 1: CLI Foundations | 4 lessons + quiz + challenge | ✅ Complete |
| Module 2: AI Integration | 2 lessons | ✅ Complete |
| Module 3: Tools & Plugins | 1 lesson | 🟡 Needs more |
| Module 4: Production | 0 lessons | 🔴 Missing |

**Claude Code — Java Track (`track-java/`):**
| Content | Status |
|---------|--------|
| All modules | 🔴 Empty (archived in `_legacy/`) |

**Microservices — Spring Boot Track (`track-spring-boot/`):**
| Module | Lessons | Status |
|--------|---------|--------|
| Module 1: Service Decomposition | 1 lesson | ✅ Complete |
| Module 2: Communication Patterns | 1 lesson | ✅ Complete |
| Module 3: Observability | 1 lesson | ✅ Complete |
| Module 4: Production | 0 lessons | 🔴 Missing |

**Microservices — NestJS Track (`track-nestjs/`):**
| Content | Status |
|---------|--------|
| All modules | 🔴 Empty (no content exists) |

### 2.3 Archived Content (`_legacy/`)

Files preserved but not yet migrated to the lesson structure:

```
curriculum/systems/claude-code/_legacy/
├── 01-architecture-overview.md     # 757 bytes — high-level arch
├── 02-key-decisions.md              # 849 bytes — design decisions
├── 03-system-diagrams.md            # 635 bytes — diagram descriptions
├── 04-implementation-guide.md       # 647 bytes — implementation patterns
├── 01-system-spec.md                # EMPTY — no content
├── 01-tradeoffs.md                  # EMPTY — no content
├── 01-java-setup.md                 # Java track setup (from old structure)
├── 01-python-setup.md               # Python setup (from old structure)
├── 01-challenge.md                  # Old challenge (different from current)
└── 01-quiz.md                       # Old quiz (superseded by new quizzes)
```

### 2.4 Knowledge Base — All Active

| Domain | Files | Content |
|--------|-------|---------|
| **Principles** | 9 | CAP, SOLID, DRY, KISS, YAGNI, ACID, BASE, CQRS, Circuit Breaker |
| **Patterns** | 6 | Factory, Singleton, Observer, Strategy, Adapter, Decorator |
| **Tools** | 5 | Docker, Kubernetes, Terraform, Git, CI/CD |
| **Technologies** | 5 | Kafka, Redis, PostgreSQL, gRPC, GraphQL |

All knowledge base files have real content. Cross-referencing via `knowledge_refs` in lesson frontmatter is implemented.

### 2.5 Search Resources

| Tag | Content |
|-----|---------|
| `java.json` | YouTube, websites, articles, courses, books, tools |
| `system-design.json` | YouTube, websites, articles, courses, books |

---

## 3. CLI — CURRENT STATE

### 3.1 Commands (17 total)

| Command | Status | Notes |
|---------|--------|-------|
| `index` (default) | ✅ | Interactive dashboard with BUILD/QUIZ/DOCTOR/RESOURCES tabs |
| `auth` | ✅ | GitHub auth status + logout |
| `login` | ✅ | GitHub PKCE OAuth flow (opens browser) |
| `logout` | ✅ | Clear cached auth token |
| `init` | ✅ | Scaffold project + review package |
| `list` | ✅ | List systems + details |
| `quiz` | ✅ | Interactive quiz runner |
| `doctor` | ✅ | Environment tool checker |
| `validate` | ✅ | **Updated — integrated executor system** |
| `verify` | ✅ | Verify against specification |
| `submit` | 🟡 | Packages locally — **NO PR automation** |
| `progress` | ✅ | Local progress tracking |
| `resources` | ✅ | Curated resource viewer |
| `challenge` | 🟡 | Challenge viewer (no auto-testing) |
| `update` | 🟡 | Update checker stub |
| `search` | 🟡 | Search stub |
| `verify` | 🟡 | Verify stub |

### 3.2 Executor Plugin System (10 files — NEW)

| Executor | Type | Status |
|----------|------|--------|
| FileExists | `file-exists` | ✅ |
| Http | `http` | ✅ |
| FileContains | `file-contains` | ✅ |
| Regex | `regex` | ✅ |
| NpmTest | `npm-test` | ✅ |
| CliCommand | `cli-command` | ✅ |
| Docker | `docker` | ✅ |
| Registry | Auto-registration | ✅ |
| Runner | `runLessonValidators` | ✅ |
| Index | Barrel exports | ✅ |

### 3.3 Readers (5)

| Reader | Purpose | Status |
|--------|---------|--------|
| `system-reader` | System discovery + metadata | ✅ |
| `quiz-reader` | Quiz file parsing | 🟡 Doesn't scan lesson dirs |
| `spec-reader` | Specification parsing | ✅ |
| `resource-reader` | Resource link parsing | ✅ |
| `index` | Core helpers + frontmatter parser | ✅ |

### 3.4 Auth

| Method | Status |
|--------|--------|
| PKCE loopback (browser) | ✅ |
| Device Flow (headless) | ✅ |
| Token caching (`~/.100x/auth.json`) | ✅ |

---

## 4. WEBSITE — CURRENT STATE

### 4.1 Pages & Routes

| Route | Component | Status |
|-------|-----------|--------|
| `/` | Home page (hero, philosophy, systems, open source) | ✅ |
| `/systems` | Systems listing grid | ✅ |
| `/systems/[slug]` | System detail (folder tags OR tracks) | 🟡 Needs lesson navigation |
| `/systems/[slug]/read/[...path]` | Reading page with lesson-aware nav | ✅ **Updated** |
| `/systems/[slug]/folder/[...path]` | Folder browsing | ✅ |
| `/principles` | Knowledge domain listing | ✅ |
| `/patterns` | Knowledge domain listing | ✅ |
| `/tools` | Knowledge domain listing | ✅ |
| `/technologies` | Knowledge domain listing | ✅ |
| `/languages` | Language listing | ✅ |
| `/search` | Search page | ✅ |

### 4.2 Design System

| Layer | Components | Status |
|-------|------------|--------|
| **Tokens** | Colors, Typography, Radius, Spacing, Shadows, Icons | ✅ |
| **Atoms** | Button, Input, Badge, Tag, Spinner, Heading, Text, Select, Toggle, Progress, Skeleton, Icon | ✅ |
| **Composites** | Breadcrumbs, Accordion, Alert, SearchInput, TabBar, Pagination, DataGrid, CodeBlock, Timeline, Cards | ✅ |
| **Layout** | SidebarNav, Header, MobileNav, Footer, Dropdown | ✅ |
| **Animations** | KineticText, CoolMode, RippleButton, NumberTicker, BlurFade, WordRotate, etc. | ✅ |

### 4.3 Curriculum Reader (`mdx.ts`)

| Function | Purpose | Status |
|----------|---------|--------|
| `getAllSystems` | System listing | ✅ |
| `getSystemMeta` | System metadata | ✅ |
| `getSystemFolderTags` | Old folder_tag structure | ✅ **Updated — excludes tracks** |
| `getSystemTracks` | NEW — lesson tracks | ✅ |
| `getTrackModules` | NEW — track modules | ✅ |
| `getModuleLessons` | NEW — module lessons | ✅ |
| `getAllSystemLessons` | NEW — flattened lessons | ✅ |
| `getLessonByPath` | NEW — lesson URL resolution | ✅ |
| `getLessonNavigation` | NEW — prev/next nav | ✅ |
| `systemHasTracks` | NEW — track detection | ✅ |
| `getSystemFileAtPath` | Updated — lesson fallback | ✅ |
| `getSystemFile` | Updated — lesson fallback | ✅ |
| `getSystemFlatFiles` | Updated — excludes track dirs | ✅ |
| `getKnowledgeItems` | Knowledge base | ✅ |

---

## 5. SUBMISSIONS — CURRENT STATE

| Feature | Status |
|---------|--------|
| Review package creation | ✅ |
| Metadata JSON generation | ✅ |
| Local submissions/ folder storage | ✅ |
| Progress tracking on submit | ✅ |
| GitHub PR automation (Octokit) | 🔴 Missing |
| Certificate generation workflow | ✅ **Created** |
| Community submissions viewer | 🔴 Missing |
| Reviewer PR labels | 🔴 Missing |

---

## 6. INFRASTRUCTURE

| Feature | Status |
|---------|--------|
| GitHub Actions publish | ✅ |
| GitHub Actions certificates | ✅ **In submissions repo** |
| Vercel deployment | ✅ |
| GitHub Pages for certificates | 🟡 Not enabled yet |
| Supabase client (unused) | 🟡 Present but not integrated |
| Google AdSense | ✅ In layout |

---

## 7. GAPS & DEAD CODE

### 7.1 Cleanup Status

After audit and cleanup:

| Item | Status | Notes |
|------|--------|-------|
| Old curriculum folder_tag structure | ✅ Archived to `_legacy/` | Content preserved for migration |
| Empty track/module directories | ✅ Removed | No content existed |
| 0-byte files (`system-spec.md`, `tradeoffs.md`) | ✅ Deleted | No content to preserve |
| Old quiz file (superseded) | ✅ Deleted | Content migrated to new quiz |
| `submissions/.gitkeep` | ✅ Removed | Redundant with README.md |
| `website/src/infrastructure/supabase.ts` | ✅ VERIFIED — NOT dead | Imported by 5 service files (communityService, achievementsService, analyticsService, progressService, profilesService). ⚠️ Note: these service files themselves may be unused in the website — verified as not orphaned, but their consumers need further audit. |
| `cli/src/commands/update.tsx` (2706 bytes) | ✅ VERIFIED — NOT dead | Substantial file with real implementation |
| `cli/src/commands/search.tsx` (3819 bytes) | ✅ VERIFIED — NOT dead | Real implementation |
| `cli/src/commands/challenge.tsx` (5384 bytes) | ✅ VERIFIED — NOT dead | Real implementation |
| `cli/src/commands/verify.tsx` (5550 bytes) | ✅ VERIFIED — NOT dead | Real implementation |

**Certificate workflow location note:** The workflow lives at `submissions/.github/workflows/certificate.yml`, which is *inside the monorepo*. For the intended architecture (separate `100xsystems/submissions` repo handling PRs independently), this workflow must be copied/moved to that separate repo's root. Currently it will only trigger on PRs to this monorepo — NOT on PRs to the submissions repo.

### 7.2 Empty Directories (Removed)

All empty directories from old structure have been cleaned up and moved to `_legacy/`.

---

## 8. INTEGRATION HEALTH

```
CLI (runtime)  ←→  Curriculum/ (filesystem)  ←→  Website (build time)

CLI:   reads curriculum at RUNTIME     → works ✅
Website: reads curriculum at BUILD TIME  → works ✅
CLI:   curriculum tracks → executors   → works ✅
CLI:   validate → executor plugins     → works ✅
CLI:   submit → GitHub PR              → ❌ Not automated
Website: lesson nav → prev/next        → works ✅
Website: breadcrumbs → track/module    → works ✅
Website: old folder_tag structure      → works ✅ (backward compat)
```
