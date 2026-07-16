# 100xSystems — What's Next: Features, Ideas & Possibilities

> **Built on:** Executor plugin system + Lesson-based curriculum + PR-based review
> **Constraint:** All features must work without a traditional backend (GitHub is the backend)

---

## P0 — CRITICAL (Foundation Gaps)

### 1. GitHub PR Submission Automation

**The single most impactful missing feature.** Everything else builds on this.

**What:** `100x submit` should fork `100xsystems/submissions`, clone it, copy the review package, commit, push, and create a PR via Octokit.

**Why it's P0:** Without this, the entire review workflow is theoretical. Users must manually create PRs. This is the bottleneck blocking all community engagement.

**Implementation:**
```
cli/src/actions/submit-pr.ts   ← NEW (Octokit-based)
cli/src/actions/submit.ts      ← UPDATE (call submit-pr)
```

**Dependencies:** `npm install @octokit/action` (already have GitHub auth)

### 2. Quiz Reader — Lesson Directory Scanning

**What:** The `quiz-reader.ts` only looks in `quizzes/` folder. Update it to also scan lesson directories for `quiz.md` files.

**Why:** Quizzes are now embedded in lesson modules. The `100x quiz` command can't find them.

**Implementation:** Add `findQuizzesInLessons()` to `quiz-reader.ts`

### 3. System Detail Page — Track/Module/Lesson Navigation

**What:** The system detail page (`/systems/[slug]`) still shows old `folderTag` structure. Systems with tracks should show track → module → lesson hierarchy instead.

**Implementation:** Update `SystemDetailClient.tsx` to call `getSystemTracks()` when `systemHasTracks()` returns true.

---

## P1 — HIGH VALUE (Community + Engagement)

### 4. Community Submissions Gallery Page

**What:** A `/community` page showing verified implementations. Read from GitHub API (`100xsystems/submissions`) at build time.

**Why:** Social proof drives engagement. Seeing other people's implementations motivates learners.

**Implementation:**
- Fetch submissions directory via GitHub API during build
- Group by system, show author, language, PR link
- Use Incremental Static Regeneration (ISR) to stay fresh
- One GitHub API call per build (zipped repo fetch)

**Data shown:**
```
Microservices
  ✓ @aryan — TypeScript — PR #42
  ✓ @alice — Spring Boot — PR #38
  Reviewers: @sarah, @bob
```

### 5. Reviewer Leaderboard (from GitHub data)

**What:** Show top reviewers based on PR reviews in `100xsystems/submissions`.

**Why:** Gamification for reviewers. People contribute more when their work is visible.

**Implementation:** GitHub API can return PR review counts. Aggregate at build time.

### 6. VS Code Extension — Phase 1: Foundation

**What:** A VS Code extension that wraps the CLI into a graphical experience inside the editor. Sidebar with system/module/lesson tree, button-based commands, GitHub auth integration.

**Why:** Developers spend 8+ hours/day in VS Code. Every context switch to a browser or terminal costs ~15 minutes of focus recovery. The extension removes this friction entirely. See `docs/VSCE_EXTENSION_AUDIT.md` for the complete analysis.

**Implementation:**
- CLI Bridge module (`child_process.spawn` calling `100x` commands with `--json` output)
- `TreeDataProvider` sidebar showing systems → tracks → modules → lessons with progress
- 4 commands: Validate, Init, Doctor, Show Next Lesson
- VS Code native GitHub auth (`authentication.getSession('github')`) synced to CLI's `~/.100x/auth.json`
- ~1000 lines, 1-2 weeks, 1 developer

**Why P1:** Highest ROI per line of code. Ambient sidebar presence drives retention better than any other feature. No competitor has a good VS Code learning extension — first-mover opportunity.

**Why not P0:** Phase 1 works independently of P0 features (sidebar, validate, doctor, init all work without PR automation or quiz scanning). The Submit button comes in Phase 3, which DOES depend on P0's PR automation.

**⚠️ Prerequisite:** Before Phase 1 can start, CLI commands need `--json` output flags. The CLI is built with Pastel/Ink (React terminal rendering) and currently outputs only ANSI text. Adding machine-readable `--json` output requires refactoring each command to separate data from presentation. Estimated: 8-10 days for the 5 commands needed by Phase 1. See `docs/VSCE_EXTENSION_AUDIT.md` Section 3.2 for details.

### 7. Knowledge Base — "Related" Section on Reading Pages

**What:** When reading a lesson, show linked knowledge base entries in a sidebar section.

**Why:** Lessons already have `knowledge_refs` in frontmatter. The website doesn't display them.

**Implementation:** Read `knowledge_refs` from lesson frontmatter, fetch KB items, render as links in sidebar.

---

## P2 — PRODUCT-MARKET FIT (Growth + Retention)

### 8. CLI Command: `100x review` — AI-Powered Review

**What:** Run an LLM evaluation of the user's implementation against the system specification.

**Why:** Instant feedback before human review. The architecture doc describes this as a signature feature.

**Implementation:**
```yaml
# In lesson frontmatter
review_criteria:
  - category: "Architecture"
    questions:
      - "Are service boundaries well-defined?"
      - "Is there a clear separation of concerns?"
```
The CLI sends the review package + criteria to an LLM and gets structured feedback.

### 9. Certificate Verification Pages

**What:** After the GitHub Actions workflow generates a certificate HTML, link to it from the website. Add a `/verify/[cert-id]` page.

**Why:** Certificates are only meaningful if they're verifiable. A public verification URL proves authenticity.

**Implementation:** The workflow already generates HTML. The website just needs a route that reads from the submissions repo's `docs/certificates/`.

### 10. CLI Command: `100x learn` — Daily Guided Path

**What:** Shows the next uncompleted lesson across all systems, with estimated time and difficulty.

**Why:** Reduces "what should I do next?" friction. Keeps learners engaged.

**Implementation:** Combine progress tracking + curriculum reading. Already have both pieces.

### 11. Homepage — "Continue Learning" Section

**What:** Show recently accessed systems and next lesson when a GitHub-authenticated user visits.

**Why:** Reduces friction. "Welcome back, finish Lesson 4 of Claude Code."

**Implementation:** Store last-read lesson in local storage. Display on homepage.

---

## P3 — DIFFERENTIATION (Unique Value)

### 12. Multi-Language Lesson Comparison

**What:** For systems with multiple tracks, show a side-by-side lesson comparison.

**Why:** Unique value proposition — "See how the same system is built in Java vs TypeScript."

**Implementation:** When viewing a lesson, if the same module exists in another track, show a toggle/link.

### 13. Engineering Decision Log Generator

**What:** After completing a system, the CLI generates an Engineering Decision Log from git history + lesson criteria.

**Why:** The architecture doc identifies this as the core "proof of engineering thinking."

**Implementation:** Parse git log for decision-related commits, combine with lesson criteria, generate `design/decisions.md`.

### 14. Interactive Architecture Diagrams

**What:** Replace static text diagrams with interactive Mermaid/D2 diagrams that respond to hover/click.

**Why:** Systems architecture is visual. Static ASCII diagrams are insufficient for complex topologies.

**Implementation:** Use `mermaid` library in the website's markdown renderer. Add a custom MDX component.

### 15. Docker-in-CLI Sandboxed Validators

**What:** For lessons that build Docker images, run containers, or need isolated testing, executors should spin up Docker containers.

**Why:** Some validators (build, test, deploy) need isolation. The `docker` executor exists but runs on the host.

**Implementation:** The docker executor should create ephemeral containers for testing, not run commands on the host.

### 16. "Build in Public" Mode

**What:** Optionally publish progress/build logs to a public GitHub Gist or issue.

**Why:** Builds personal brand for learners. "I'm building Claude Code live — follow my progress."

**Implementation:** After each `100x validate` or `100x submit`, push a progress update to a gist.

---

## P4 — EXPANSION (Scale)

### 17. Mobile Apps (iOS + Android)

**What:** React Native / Flutter apps consuming the same curriculum.

**Why:** The architecture is content-centric. Any client can read the curriculum.

**Implementation:** Create shared API layer or use the existing file system structure. Mobile apps read from curriculum repo directly.

### 18. Curriculum Contribution Workflow

**What:** Streamlined process for community members to contribute new systems, tracks, modules, or lessons.

**Why:** Growth depends on content. Open-source contributions scale content creation.

**Implementation:**
```
100x contribute init {system}
→ Creates lesson template with frontmatter
→ Generates validator stubs
→ Opens PR to 100xsystems/curriculum
```

### 19. Analytics Dashboard (Privacy-First)

**What:** Show system popularity, completion rates, and common failure points.

**Why:** Data-driven curriculum improvements. Identify which lessons need better content.

**Implementation:** Anonymous opt-in analytics. No PII. Track: lesson viewed, quiz score, submit attempt.

### 20. System Dependencies Graph

**What:** Visual graph showing prerequisite chains between systems and knowledge base entries.

**Why:** "You need to understand CAP Theorem before tackling Microservices." Visual learning paths reduce confusion.

**Implementation:** Parse `prerequisites` and `knowledge_refs` from all lessons. Build a dependency graph. Render with D3.js.

---

## P5 — NICE-TO-HAVE (Polish + Delight)

### 21. CLI Themes & Customization

**What:** Custom colors, fonts, and layouts for the Ink-based CLI.

### 22. Achievement Badges (in GitHub Profile)

**What:** After completing a system, add a badge to the user's GitHub profile README via GitHub Actions.

### 23. "Explain This System" — AI Summary

**What:** One-command AI-generated summary of any system: `100x explain microservices`

### 24. Offline Mode

**What:** Full CLI functionality without internet (except LLM calls). Curriculum is local.

### 25. Import/Export Progress

**What:** Share progress between machines. `100x progress export` → JSON file.

---

## ARCHITECTURAL CONSTRAINTS SUMMARY

| Constraint | Why |
|------------|-----|
| No traditional backend | Cost, maintenance, scaling |
| GitHub is the database | PRs, reviews, metadata all on GitHub |
| Curriculum is the single source of truth | CLI and website read the same files |
| No JSON metadata files | Everything in frontmatter — rename-safe |
| Executor plugins, not system-specific logic | CLI never needs to know about specific systems |
| Static site generation | Website rebuilds on curriculum changes |
| Certificates are static HTML | GitHub Actions generates, GitHub Pages serves |
