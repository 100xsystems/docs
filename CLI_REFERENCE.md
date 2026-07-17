# 100xSystems CLI — Complete Command Reference & Testing Guide

> **CLI Version:** 0.1.0 | **Framework:** Pastel + Ink (React for CLIs) | **Language:** TypeScript ESM
> **Entry point:** `cli/dist/index.js` | **Build:** `cd cli && npm run build`
> **Register for testing:** `cd cli && npm link` (or run directly: `node cli/dist/index.js <command>`)

---

## Table of Contents

| # | Command | Category | Recent? |
|---|---------|----------|---------|
| 0 | [`100x` (Dashboard)](#0-100x--dashboard) | Navigation | ✅ New |
| 1 | [`100x list`](#1-100x-list) | Discovery | |
| 2 | [`100x search`](#2-100x-search) | Discovery | |
| 3 | [`100x init`](#3-100x-init) | Project | |
| 4 | [`100x validate`](#4-100x-validate) | Verification | |
| 5 | [`100x verify`](#5-100x-verify) | Verification | |
| 6 | [`100x review`](#6-100x-review--ai-powered) | Verification | ✅ **New** |
| 7 | [`100x submit`](#7-100x-submit) | Submission | |
| 8 | [`100x learn`](#8-100x-learn) | Learning | ✅ **New** |
| 9 | [`100x quiz`](#9-100x-quiz) | Learning | |
| 10 | [`100x challenge`](#10-100x-challenge) | Learning | |
| 11 | [`100x progress`](#11-100x-progress) | Tracking | |
| 12 | [`100x public`](#12-100x-public--build-in-public) | Social | ✅ **New** |
| 13 | [`100x contribute`](#13-100x-contribute) | Contribution | ✅ **New** |
| 14 | [`100x doctor`](#14-100x-doctor) | Environment | |
| 15 | [`100x resources`](#15-100x-resources) | Learning | |
| 16 | [`100x login`](#16-100x-login) | Auth | |
| 17 | [`100x auth`](#17-100x-auth) | Auth | |
| 18 | [`100x logout`](#18-100x-logout) | Auth | |
| 19 | [`100x update`](#19-100x-update) | Maintenance | |
| 20 | [`100x contribute track\|lesson`](#20-sub-commands-contribute-track--contribute-lesson) | Contribution | ✅ **New** |

---

## Setup: Build & Link

```bash
# Build the CLI
cd /Users/aryanbatra/IdeaProjects/100xsystems/cli
npm run build

# Register globally (uses local build instead of npm package)
npm link

# Verify it works
100x --version   # Should show 0.1.0
100x doctor      # Should run local version
```

> ⚠️ **Whenever you edit CLI source files, rebuild**: `npm run build` then retest.

---

## 0. `100x` — Dashboard

**File:** `cli/src/commands/index.tsx`
**Action file:** None (self-contained)
**Args:** None | **Options:** None

### Behavior

Opens an interactive tabbed Dashboard (inspired by OpenCode). If the user navigates via a dashboard button, it prints `100x <command>` and exits (user must then run that command manually — Pastel limitation).

### What It Renders

```
╔══ 100x SYSTEMS ENGINE ═══════════════════════╗
║ [Tab] modes · [Ctrl+M] menu                 ║
║                                              ║
║ ┌BUILD┐ ┌QUIZ┐ ┌DOCTOR┐ ┌RESOURCES┐         ║
║                                              ║
║ ┌─ Workspace ──────────┐ ┌─ Tab Content ──┐ ║
║ │ Systems: 2           │ │                │ ║
║ │ Quizzes: available   │ │                │ ║
║ │ Templates: TS/Java   │ │                │ ║
║ └──────────────────────┘ └────────────────┘ ║
║                                              ║
║ 100x ❯ _                                     ║
╚══════════════════════════════════════════════╝
```

### Testing

```bash
100x
```

**Check:**
- ✅ Dashboard loads with BUILD/QUIZ/DOCTOR/RESOURCES tabs
- ✅ `Tab` key switches between tabs
- ✅ `Ctrl+M` opens command palette (if implemented)
- ✅ Quit with `Ctrl+C` or `Esc`

---

## 1. `100x list`

**File:** `cli/src/commands/list.tsx`
**Reader:** `system-reader.ts` (`getAllSystems`, `getSystemMeta`, `getSystemFolderTags`)
**Args:** `[systemSlug?]` (optional string) | **Options:** None

### Behavior

- **Without args:** Lists all available systems from `curriculum/systems/`. Each system shows: title, description, difficulty (color-coded), tags, and suggested commands.
- **With system slug:** Shows detailed view of one system including its folder tags (quizzes, challenges, specification, implementation) with child items.

### What It Renders

**Without args:**
```
  100xSystems — Available Systems

  Claude Code
  Learn to build and extend the Claude Code CLI
  Intermediate · AI · CLI · Tool Building
  100x list claude-code
  → see sections
  100x init claude-code
  → start building

  Microservices
  Design, build, and deploy a microservices architecture
  Advanced · Distributed Systems · Architecture
  100x list microservices
  → see sections
  100x init microservices
  → start building
```

**With slug (`100x list claude-code`):**
```
  Claude Code
  Learn to build and extend the Claude Code CLI
  AI · CLI · Tool Building

  Quiz (2 items)
  📄 claude-code-fundamentals

  Specification (1 item)
  📄 SPECIFICATION.md
  100x verify → verify your implementation

  Implementation (1 item)
  100x init claude-code --lang <language> → scaffold project
```

### Testing

```bash
# List all systems
100x list

# View details for a specific system
100x list claude-code
100x list microservices

# Non-existent system
100x list nonexistent
```

**Check:**
- ✅ Without args: all systems listed with title, description, difficulty color, tags
- ✅ With valid slug: folder tags shown with child items and suggested commands
- ✅ Color coding: Advanced=red, Intermediate=yellow, Beginner=green
- ✅ With invalid slug: "System not found" message
- ✅ Empty curriculum: "No systems found" message

---

## 2. `100x search`

**File:** `cli/src/commands/search.tsx`
**Readers:** `system-reader.ts`, `resource-reader.ts`
**Args:** `[query]` (string, required) | **Options:** None

### Behavior

Searches systems by title, description, and tags. Also searches resource items by title and description. Results are limited to 30. Shows match type and suggested command for system results.

### What It Renders

```
  Search results for "database" (2 found)

  ⚡ PostgreSQL
    Tags match "database"
    100x init postgresql → start building

  📚 Database Scaling
    In Microservices > Papers
```

### Testing

```bash
# Search by name
100x search claude

# Search by tag
100x search distributed

# Search by topic in resources
100x search kafka

# No results
100x search xyznonexistent
```

**Check:**
- ✅ Matches by title, description, AND tags
- ✅ Resource matches shown with 📚 icon, system matches with ⚡
- ✅ "N results found" count shown
- ✅ Limited to 30 results with overflow message
- ✅ Empty query: "Please provide a search query" message
- ✅ No results: "No results for X" message

---

## 3. `100x init`

**File:** `cli/src/commands/init.tsx`
**Actions:** `scaffold/index.ts` (`scaffoldProject`), `actions/progress.ts` (`markInProgress`)
**Readers:** `system-reader.ts`, `spec-reader.ts`, `auth/index.ts`
**Args:** `[systemSlug]` (string, required)
**Options:**
| Flag | Alias | Values | Default |
|------|-------|--------|---------|
| `--lang` | `-l` | `typescript` \| `java` | `typescript` |
| `--output` | `-o` | Any path | `./<slug>-implementation` |
| `--author` | `-a` | GitHub username | Cached user or empty |

### Behavior

1. Validates the system exists
2. Resolves output directory (checks it's empty)
3. Reads specification from curriculum
4. Gets cached GitHub user for author template
5. Calls `scaffoldProject()` which creates all files
6. Calls `markInProgress()` to track the project
7. Checks for quizzes/challenges and suggests them
8. Renders a summary of created files

### What It Renders

```
  100xSystems — Initializing "Claude Code"

  System:   Claude Code
  Output:   ./claude-code-implementation
  Language: typescript

  ✓ Project created successfully!

  Documentation:
    📝 README.md
    📝 design/decisions.md
    📝 design/architecture.md
    📝 design/tradeoffs.md
    📝 verification/checklist.md

  Code:
    📄 src/index.ts
    📄 src/package.json
    📄 src/tsconfig.json

  Config:
    ⚙️  .100x.json

  Next steps:
  cd ./claude-code-implementation
  100x validate → check document completeness
  100x verify   → check against specification

  Also try:
  100x quiz claude-code → take quizzes

  When you're ready to submit your implementation:
  → Run 100x submit to prepare your review package
```

### Testing

```bash
# Basic init
100x init claude-code

# Init to specific directory
100x init claude-code --output /tmp/my-claude

# Init with Java
100x init claude-code --lang java

# Init with custom author
100x init microservices --author aryanbatras --output /tmp/ms

# Init again to same location (should fail)
100x init claude-code
```

**Check:**
- ✅ Created directory structure has: `.100x.json`, `README.md`, `design/`, `verification/`, `src/`, `specification/`
- ✅ `.100x.json` has correct system, systemTitle, language fields
- ✅ Design docs have templates with proper sections
- ✅ Progress tracked: `100x progress` shows the new project
- ✅ Error if system doesn't exist
- ✅ Error if output directory exists and is non-empty
- ✅ Language-specific templates (TypeScript vs Java)

---

## 4. `100x validate`

**File:** `cli/src/commands/validate.tsx`
**Actions:** `actions/validate.ts` (`runValidation`)
**Scaffold:** `scaffold/index.ts` (`readProjectConfig`)
**Args:** None | **Options:** None

### Behavior

1. Checks for `.100x.json` in current directory
2. Calls `runValidation()` which runs: documentation checks (README, decisions, architecture, tradeoffs, checklist), structure checks (.100x.json, src/, design/), git checks
3. Renders results using the `ValidationReport` UI component

### What It Renders

```
  100xSystems — Validating "Claude Code"

  Documentation
  ✓ README.md exists with content
  ✓ design/decisions.md has proper decision log format
  ⚠ design/architecture.md exists but may be incomplete
  ✓ design/tradeoffs.md acknowledges trade-offs
  ✓ verification/checklist.md exists

  Structure
  ✓ .100x.json project config found
  ✓ Source code directory with 1 file(s)
  ✓ design/ directory with 4 file(s)

  ──────────────────────────────────────────
  Validation Results: 8 passed, 0 failed
  Your project is ready for submission!
```

### Testing

```bash
# From inside a project directory
cd /tmp/my-claude
100x validate

# From outside a project (no .100x.json)
cd /tmp
100x validate
```

**Check:**
- ✅ All checks displayed with ✓ (pass), ⚠ (warn), ✗ (fail)
- ✅ Summary count at bottom
- ✅ Failed checks show hints for fixing
- ✅ "Ready for submission" message when all pass
- ✅ Error message if no `.100x.json` found
- ✅ Documentation checks: README.md (≥50 chars), decisions.md (Context+Decision sections), architecture.md (≥100 chars), tradeoffs.md (≥100 chars), checklist.md

---

## 5. `100x verify`

**File:** `cli/src/commands/verify.tsx`
**Actions:** `actions/verify.ts` (`runSpecCheck`), `actions/validate.ts` (`checkDocumentation`, `checkStructure`)
**Reader:** `spec-reader.ts` (`getSpec`)
**Args:** None | **Options:** None

### Behavior

1. Checks for `.100x.json` in current directory
2. Reads specification from curriculum (`getSpec(systemSlug)`)
3. Runs spec-defined checks (file-exists, doc-section, test-passes, docker-build, etc.)
4. Runs documentation completeness checks (same as `validate`)
5. Runs structure checks (same as `validate`)
6. Shows summary with pass/fail/skip counts

### What It Renders

```
  100xSystems — Verifying "Claude Code"
  Specification: Claude Code Architecture (v1.0)
  Project: /tmp/my-claude

  Specification Checks
  ✓ file-exists design/decisions.md
  ✓ doc-section README.md Technical Overview
  ✓ test-passes npm test

  Documentation Completeness
  ✓ README.md exists with sufficient content
  ✓ Engineering Decision Log exists with sufficient content
  ...

  Project Structure
  ✓ .100x.json project config found
  ✓ Source code directory (src/) found
  ...

  ──────────────────────────────────────────
  Results: 12 passed, 0 failed
  All checks passed! Your implementation looks good.
```

### Testing

```bash
# From inside a project directory
cd /tmp/my-claude
100x verify

# From outside a project
cd /tmp
100x verify
```

**Check:**
- ✅ Shows spec title, version, and project path
- ✅ Spec checks displayed (from curriculum spec YAML)
- ✅ Documentation + structure checks displayed
- ✅ Pass/fail/skip counts in summary
- ✅ Failure hints shown for failed checks
- ✅ "All checks passed!" vs "N check(s) failed" message
- ✅ Error if no `.100x.json`

---

## 6. `100x review` — AI-Powered (✅ NEW)

**File:** `cli/src/commands/review.tsx`
**Actions:** `actions/review.ts` (`runReview`)
**Scaffold:** `scaffold/index.ts` (`readProjectConfig`)
**Reader:** `lesson-reader.ts` (`getAllSystemLessons`, `getLessonReviewCriteria`)
**Args:** `[systemSlug?]` (optional string — auto-detected from project)
**Options:**
| Flag | Alias | Description |
|------|-------|-------------|
| `--api-key` | `-k` | OpenAI API key (falls back to `OPENAI_API_KEY` env) |
| `--api-base` | `-b` | Custom API base URL (for local models like Ollama, LM Studio) |
| `--model` | `-m` | Model name (default: `gpt-4o-mini`) |
| `--lesson` | `-l` | Review only against a specific lesson's criteria |

### Behavior

1. Resolves system slug (from arg or `.100x.json`)
2. Collects review package: README.md, design/decisions.md, design/architecture.md, design/tradeoffs.md, verification/checklist.md, specification/SPECIFICATION.md, up to 10 source files
3. Reads review criteria from lesson frontmatter (`review_criteria` field). Falls back to defaults (Architecture, Design Decisions, Code Quality, Documentation) if none found.
4. Builds a structured prompt and sends it to the LLM API
5. Parses the JSON response into per-category scores, strengths, weaknesses
6. Renders a color-coded report

### What It Renders

```
  100xSystems — AI Engineering Review

  Claude Code · Reviewed 7/15/2026, 2:30 PM

  Overall Score: 82/100 (4 strengths · 2 weaknesses)

  Architecture | 85/100
    Well-structured with clear separation of concerns.
    ✓ Clear service boundaries · Well-documented
    ⚠ No async communication pattern described

  Design Decisions | 78/100
    Decisions are justified but alternatives not fully explored.
    ✓ Database choice justified
    ⚠ No comparison with alternative architectures

  Score: 80-100 Excellent · 50-79 Needs work · 0-49 Missing/Incomplete
```

### Testing

```bash
# From inside a project with OPENAI_API_KEY set
cd /tmp/my-claude
export OPENAI_API_KEY=sk-...
100x review

# Specify system and use a local model
100x review claude-code --api-base http://localhost:1234/v1 --model llama3

# Review only specific lesson
100x review --lesson lesson-3-deployment

# Without API key (should show clear error)
100x review claude-code
```

**Check:**
- ✅ Default criteria shown when lesson has no `review_criteria` frontmatter
- ✅ Per-category scores (0-100) with color coding
- ✅ Strengths (green ✓) and weaknesses (yellow ⚠) per category
- ✅ Overall score computed as average of all categories
- ✅ Error handling: missing API key → clear message
- ✅ Error handling: bad API key → HTTP error shown
- ✅ Error handling: no `.100x.json` and no slug → clear message
- ✅ Works with local models (Ollama, LM Studio)

---

## 7. `100x submit`

**File:** `cli/src/commands/submit.tsx`
**Actions:** `actions/submit.ts`, `actions/submit-pr.ts`
**Scaffold:** `scaffold/index.ts` (`readProjectConfig`)
**Args:** `[systemSlug?]` (optional string) | **Options:** None

### Behavior (Multi-Step Wizard)

**Phase 1 — Loading:** Reads `.100x.json`, resolves system slug
**Phase 2 — Confirm:** Runs validation, shows results, asks "Submit for review?"
**Phase 3 — Auth:** Opens browser for GitHub OAuth (PKCE flow)
**Phase 4 — Metadata:** Interactive form for: repository URL, implementation language (select), difficulty (text input)
**Phase 5 — Building:** Collects review package, copies docs, generates metadata JSON
**Phase 6 — Creating PR:** Forks `100xsystems/submissions`, clones, copies files, commits, pushes, creates PR
**Phase 7 — Done:** Shows PR URL (or manual instructions if PR fails)

### What It Renders

**Auth step:**
```
  Step 1/3: Authenticating with GitHub...
    A browser window will open for GitHub authorization.
```

**Metadata step:**
```
  Step 2/3: Gathering submission metadata
  Link to your implementation repository:
  > https://github.com/your-username/your-repo
```

**Done (with PR):**
```
  ✓ Pull Request created successfully!

  → https://github.com/100xsystems/submissions/pull/42

  A reviewer will review your submission. Track the PR for updates.

  Submission details:
  System:    claude-code
  Author:    aryanbatras
  Language:  typescript
  Repository: https://github.com/aryanbatras/my-implementation
  PR: #42
```

### Testing

```bash
# From inside a project
cd /tmp/my-claude
100x submit

# With explicit system slug
100x submit claude-code

# From outside a project
cd /tmp
100x submit
```

**Check:**
- ✅ Phase 1: Loading state with spinner
- ✅ Phase 2: Validation results + confirm prompt
- ✅ "Submission cancelled." when user says no
- ✅ Phase 3: GitHub auth opens browser
- ✅ Phase 4: Repository URL, language select, difficulty text input
- ✅ Phase 5: Building state with spinner
- ✅ Phase 6: PR creation with fork/clone/copy/commit/push
- ✅ Phase 7: PR URL shown + submission details
- ✅ Fallback: If PR fails, show manual git commands
- ✅ Auto-exit after 2 seconds on done
- ✅ Step indicator (●/◉/○) for multi-step progress

---

## 8. `100x learn` (✅ NEW)

**File:** `cli/src/commands/learn.tsx`
**Actions:** `actions/learn.ts` (`getLearnDashboard`)
**Args:** None | **Options:** None

### Behavior

1. Loads progress from `~/.100x/progress.json`
2. Scans all systems with tracks
3. Counts completed lessons per system (heuristic: checks for design docs)
4. Finds the first uncompleted lesson across all systems
5. Renders a learning dashboard with progress bar, next lesson, and system list

### What It Renders

```
  100xSystems — Learning Dashboard

  Overall Progress: 25% (2/8 lessons across 2 systems)

  ▶ Next Up
  ┌──────────────────────────────────────┐
  │ Claude Code                          │
  │  · track-typescript                  │
  │                                      │
  │ Lesson: Lesson 2 - Context Manager    │
  │ Build a context window manager that   │
  │ tracks token usage.                  │
  │                                      │
  │ Difficulty: Intermediate · Estimated: │
  │ 45 minutes                           │
  │                                      │
  │ Module: module-2-context-management   │
  │ · Track: track-typescript            │
  └──────────────────────────────────────┘

  All Systems
  ✓ Claude Code  2/4 lessons · 50%
  ○ Microservices  0/4 lessons · 0%

  Quick start: 100x list · 100x quiz <system> · 100x challenge <system>
```

### Testing

```bash
100x learn
```

**Check:**
- ✅ Overall progress percentage with fraction
- ✅ "Next Up" card with system title, track, lesson title, description, difficulty, estimated time, module, track
- ✅ Each system shows: status icon (✓/▶/○), title, lesson count, percent
- ✅ When all lessons complete: "You've completed all available lessons! 🎉"
- ✅ Works without having started any project (all systems show 0%)

---

## 9. `100x quiz`

**File:** `cli/src/commands/quiz.tsx`
**Reader:** `quiz-reader.ts` (`getQuizzes`)
**UI:** `ui/Quiz.tsx`
**Args:** `[systemSlug]` (string, required) | **Options:** None

### Behavior

1. Validates system exists
2. Reads quizzes from `curriculum/systems/[slug]/quizzes/` and embedded in lesson directories
3. Separates lesson-embedded quizzes from folder-based quizzes
4. Renders the interactive `QuizApp` component

### Testing

```bash
# Run quizzes for a system
100x quiz claude-code
100x quiz microservices

# Non-existent system
100x quiz nonexistent

# System with no quizzes
100x quiz claude-code  # (if system exists but has no quiz files)
```

**Check:**
- ✅ Shows lesson quizzes and system quizzes sections
- ✅ Interactive QuizApp component with multiple-choice and true/false
- ✅ Keyboard navigation (arrow keys, enter to submit)
- ✅ Score tracking
- ✅ Error if system not found
- ✅ Error if no quizzes found

---

## 10. `100x challenge`

**File:** `cli/src/commands/challenge.tsx`
**Actions:** `actions/challenge.ts` (`readChallenges`)
**Args:** `[systemSlug]` (string, required)
**Options:** `--start <slug>` — Start a specific challenge

### Behavior

**Without `--start`:** Lists all challenges for a system with order, title, description, difficulty, requirements, and the command to start each.

**With `--start`:** Shows detailed view of a single challenge with full description, requirements, tasks, and suggested workflow.

### What It Renders

**List view:**
```
  Claude Code Challenges

  #1. Implement a Context Manager
    Build a context window manager that tracks token usage
    Advanced
    Requirements:
      • Token counting
      • Context window management
    100x challenge claude-code --start challenge-1 → start this challenge
```

**Detail view:**
```
  Starting Challenge: Implement a Context Manager

  System:     Claude Code
  Challenge:  Implement a Context Manager
  Difficulty: Advanced

  Description:
  Build a context window manager that tracks token usage...

  Requirements:
  1. Token counting
  2. Context window management
  ...
```

### Testing

```bash
# List challenges
100x challenge claude-code

# Start a specific challenge
100x challenge claude-code --start challenge-1

# Invalid challenge slug
100x challenge claude-code --start nonexistent
```

**Check:**
- ✅ List view with numbered challenges, requirements, suggested command
- ✅ Detail view with full task breakdown
- ✅ Difficulty color coding
- ✅ Error if system not found
- ✅ Error if challenge slug doesn't exist

---

## 11. `100x progress`

**File:** `cli/src/commands/progress.tsx`
**Actions:** `actions/progress.ts` (`loadProgress`, `detectInProgressProjects`)
**Reader:** `system-reader.ts` (`getAllSystems`, `getSystemMeta`)
**Args:** `[systemSlug?]` (optional string) | **Options:** None

### Behavior

**Without args:** Shows all systems grouped by status (Completed ✓, In Progress ⟳, Not Started ○) with progress percentage.

**With slug:** Shows detailed view of one system with status, start/completed dates, project directory, language, and suggested next commands.

### What It Renders

**All systems:**
```
  100xSystems — Your Progress

  ✓ Completed
    ● Claude Code (7/15/2026)

  ⟳ In Progress
    ● Microservices
      /tmp/ms

  ○ Not Started
    ○ Cloud Code
    ○ Kafka

  ──────────────────────────────────────────
  Progress: 1/4 systems completed (25%)
  Next: 100x init microservices — Microservices
```

**Single system:**
```
  Claude Code
  Status: ✓ completed
  Started: 7/1/2026
  Completed: 7/15/2026
  Project: /tmp/my-claude
  Language: typescript
```

### Testing

```bash
# All systems
100x progress

# Specific system
100x progress claude-code

# System not yet started
100x progress microservices
```

**Check:**
- ✅ Three status groups shown with colors
- ✅ Progress percentage computed correctly
- ✅ Next suggested system shown
- ✅ Single system view shows all available data
- ✅ Shows "Status: ○ Not started" for untouched systems
- ✅ Suggested commands per status

---

## 12. `100x public` — Build in Public (✅ NEW)

**File:** `cli/src/commands/public.tsx`
**Actions:** `actions/build-in-public.ts` (`getStatus`, `initGist`, `updateGist`, `stopGist`)
**Args:** `[action?]` (optional enum: `init`, `update`, `status`, `stop`)
**Options:** None (all empty)

### Behavior

**`status` (default):** Shows whether Build in Public is active. If active: shows gist URL and last updated. If inactive: shows setup instructions.

**`init`:** Requires GitHub auth. Generates a markdown progress report from `~/.100x/progress.json`, creates a public gist via GitHub API. Saves config to `~/.100x/public-gist.json`.

**`update`:** Reads existing config, fetches latest progress, PATCHes the gist.

**`stop`:** Deletes `~/.100x/public-gist.json` (gist stays on GitHub).

### What It Renders

**Inactive:**
```
  Build in Public — Not Active

  Share your 100xSystems progress with the world!

  100x public init   — Create a public progress gist
  100x public update — Update your progress gist
```

**After init:**
```
  ✓ Build in Public Gist Created!

  Your progress gist is live at:
  https://gist.github.com/your-username/abc123

  Pin this gist to your GitHub profile to show off
  your systems engineering journey. Update it anytime:

  100x public update — Sync latest progress
```

### Gist Content (generated markdown)

```markdown
# 100xSystems — Build in Public

> Learning systems engineering by building real projects.

## Progress

**2/4** systems completed (50%)

██████████░░░░░░░░░░░░

## ✅ Completed

- [x] **Claude Code** (Intermediate)

## 🔄 In Progress

- [ ] **Microservices** — started July 1, 2026

## 📋 Up Next

- Cloud Code
- Kafka

---

_Automatically updated by 100xSystems CLI_
```

### Testing

```bash
# Check status (default)
100x public

# Check status explicitly
100x public status

# Initialize (requires GitHub auth)
100x public init

# Update
100x public update

# Stop tracking
100x public stop

# Update without init (error case)
rm -f ~/.100x/public-gist.json
100x public update
```

**Check:**
- ✅ `status`: shows active/inactive with appropriate instructions
- ✅ `init`: auth flow opens browser, gist created on GitHub
- ✅ `init`: gist has correct markdown with progress bar, sections
- ✅ `init`: config saved to `~/.100x/public-gist.json`
- ✅ `update`: gist content refreshed with latest progress
- ✅ `stop`: config file deleted (gist remains on GitHub)
- ✅ Error: `update` without `init` — clear error message

---

## 13. `100x contribute` — Curriculum Contribution (✅ NEW)

**File:** `cli/src/commands/contribute.tsx`
**Actions:** `actions/contribute.ts` (`initSystem`, `addTrack`, `addLesson`, `verifyCurriculumAccess`)
**Args:** `[action]` (enum: `init` \| `track` \| `lesson`), `[systemSlug?]`, `[languageOrModule?]`
**Options:**
| Flag | Alias | Description |
|------|-------|-------------|
| `--title` | `-t` | System title |
| `--description` | `-d` | System description |
| `--difficulty` | `-D` | Beginner, Intermediate, or Advanced |
| `--language` | `-l` | Programming language for track |
| `--tags` | `-g` | Comma-separated tags |

### Behavior

**Must be run from inside the monorepo.** Calls `verifyCurriculumAccess()` which checks that `curriculum/` directory exists.

**`init`:** Creates a full system directory structure:
- `curriculum/systems/[slug]/index.md` with YAML frontmatter
- `curriculum/systems/[slug]/track-[lang]/module-1-introduction/01-lesson-introduction-[title].md`

**`track`:** Adds a new language track to an existing system. Creates track directory + default module + lesson. Updates `index.md` tracks list.

**`lesson`:** Adds a new module+lesson to an existing track.

### What It Renders

**Init:**
```
  ✓ Curriculum content created

  Location: /Users/.../100xsystems/curriculum/systems/kafka

  Files created:
    📄 /Users/.../curriculum/systems/kafka/index.md
    📄 /Users/.../curriculum/systems/kafka/track-typescript/
    📄 /Users/.../curriculum/systems/kafka/track-typescript/module-1-introduction/
    📄 /Users/.../curriculum/systems/kafka/track-typescript/module-1-introduction/01-lesson-introduction-to-kafka.md

  Next steps:
    1. Edit the generated Markdown files with lesson content
    2. Add knowledge_refs to link to knowledge base entries
    3. Run tests to validate your lessons
    4. Commit and push your changes

  Tip: Run 100x contribute track kafka java to add another language track
```

### Generated `index.md` Example

```markdown
---
title: "Kafka"
description: "Learn to build Kafka from scratch."
difficulty: Intermediate
tags: ["kafka"]
order: 999
tracks:
  - slug: track-typescript
    title: "Typescript Track"
    language: typescript
    difficulty: Intermediate
    modules:
      - slug: module-1-introduction
        title: "Introduction"
---
```

### Testing

```bash
# Init a test system (run from monorepo root)
100x contribute init test-system --description "Test" --difficulty Beginner --tags test

# Add a language track
100x contribute track test-system java

# Add a lesson
100x contribute lesson test-system --language typescript --title "Docker" --description "Deploy with Docker"

# Verify generated files
ls curriculum/systems/test-system/
cat curriculum/systems/test-system/index.md

# Clean up
rm -rf curriculum/systems/test-system

# Outside monorepo (error case)
cd /tmp
100x contribute init foo
```

**Check:**
- ✅ `init`: Creates system directory with index.md and track/module/lesson
- ✅ `init`: index.md has YAML frontmatter (title, description, difficulty, tags, order, tracks)
- ✅ `init`: Lesson file has frontmatter (title, description, order, difficulty, estimated_time, knowledge_refs, prerequisites)
- ✅ `init`: Body has sections: Overview, Key Concepts, Implementation, Exercises, Summary
- ✅ `track`: Creates new track directory with default module+lesson
- ✅ `track`: Existing index.md updated with new track entry
- ✅ `lesson`: Creates new module+lesson with proper slug (module-N-title)
- ✅ Error: init on existing system
- ✅ Error: track on non-existent system
- ✅ Error: duplicate track
- ✅ Error: outside monorepo

---

## 14. `100x doctor`

**File:** `cli/src/commands/doctor.tsx`
**Actions:** None (self-contained — uses `execaSync`)
**Args:** `[systemSlug?]` (optional string — filters tools)
**Options:** None

### Behavior

- **Without args:** Checks all 14 tools (Node.js, Git, npm, TypeScript, Java, Maven, Docker, Docker Compose, kubectl, Terraform, AWS CLI, Python 3, Go, Rust)
- **With system slug:** Filters to tools relevant to that specific system (e.g., `claude-code` checks Node.js, Git, npm, TypeScript, Docker)
- Each tool is checked by running its version command. Required tools (Node.js, Git, npm) are highlighted if missing.

### What It Renders

```
  100xSystems — Environment Doctor

  ✓ Node.js          v26.0.0
  ✓ Git              v2.50.1
  ✓ npm              v11.12.1
  ✓ TypeScript       v5.9.3
  ✓ Java (JDK)       v17.0.19
  ○ Kubernetes (kubectl) not found (optional)
  ✓ Terraform        v1.7.0

  ──────────────────────────────────────────
  Summary:
  3 required tools OK
  Environment looks good!
  5 optional tools found
```

### System-Tool Mappings

| System | Tools Checked |
|--------|--------------|
| `claude-code` | Node.js, Git, npm, TypeScript, Docker |
| `microservices` | Node.js, Git, npm, Docker, Docker Compose |
| `event-driven` | Node.js, Git, npm, Docker |
| `kubernetes` | Node.js, Git, npm, Docker, kubectl, Docker Compose |
| `terraform` | Node.js, Git, npm, Terraform |
| `aws-infrastructure` | Node.js, Git, npm, AWS CLI, Terraform |
| `java-microservices` | Node.js, Git, npm, Java (JDK), Maven, Docker |
| `go-service` | Node.js, Git, npm, Go, Docker |
| `rust-tool` | Node.js, Git, npm, Rust (cargo) |

### Testing

```bash
# All tools
100x doctor

# System-specific
100x doctor claude-code
100x doctor kubernetes

# Non-existent system (falls back to all tools)
100x doctor nonexistent
```

**Check:**
- ✅ All tools found shown with ✓ and version
- ✅ Required tools missing: ✗ with "NOT FOUND (required)" in red
- ✅ Optional tools missing: ○ with "not found (optional)" dimmed
- ✅ Summary: required OK count + optional found count
- ✅ Missing tools listed at bottom with install suggestions
- ✅ System-specific filtering works

---

## 15. `100x resources`

**File:** `cli/src/commands/resources.tsx`
**Readers:** `resource-reader.ts`, `system-reader.ts`
**Args:** `[systemSlug?]` (optional) | **Options:** None

### Behavior

**Without args:** Lists all systems that have curated resources, with count per system.

**With slug:** Shows all resource categories and items for that system. Each item shows: type icon (📄 paper, 🎬 video, 📝 blog, 📚 documentation, 🔧 tool), title (clickable/cyan), URL, and description.

### What It Renders

```
  Claude Code Resources

  Papers
    📄 Scaling Monorepo Builds
      https://example.com/paper
      Research on build system scaling at Google

  Videos
    🎬 Building CLI Tools in Node.js
      https://youtube.com/watch?v=...
      Conference talk on CLI tooling patterns

  Documentation
    📚 Claude Code API Reference
      https://docs.example.com
```

### Testing

```bash
# List systems with resources
100x resources

# View resources for a specific system
100x resources claude-code

# System with no resources
100x resources microservices
```

**Check:**
- ✅ Without args: shows systems with resource counts
- ✅ With slug: shows categories with typed items (📄, 🎬, 📝, 📚, 🔧)
- ✅ Each item shows: title, URL, description
- ✅ Error if system not found
- ✅ Error if system has no resources

---

## 16. `100x login`

**File:** `cli/src/commands/login.tsx`
**Auth:** `auth/index.ts` (`ensureAuthenticated`, `isAuthenticated`, `getCachedUser`)
**Args:** None
**Options:** `--force` (`-f`) — Force re-authentication

### Behavior

1. Checks if already authenticated (unless `--force`)
2. If authenticated: shows user info
3. If not (or `--force`): triggers GitHub OAuth PKCE flow
4. Opens browser to auth proxy, user authorizes, token saved to `~/.100x/auth.json`

### What It Renders

```
  GitHub Authentication

  A browser window will open to authorize 100xSystems.
  If it doesn't open automatically, follow the URL shown above.
```

**Already authenticated:**
```
  Authenticated as: Aryan Batra (aryanbatras)
  Use 100x login --force to re-authenticate.
```

### Testing

```bash
# First time login
100x login

# Already authenticated
100x login

# Force re-authentication
100x login --force
100x login -f
```

**Check:**
- ✅ If already authed: shows user name and login
- ✅ If not authed: opens browser for OAuth
- ✅ `--force`: re-authenticates even if already logged in
- ✅ Error handling: failed auth shows error message
- ✅ Success: "✓ Authenticated successfully as X"

---

## 17. `100x auth`

**File:** `cli/src/commands/auth.tsx`
**Auth:** `auth/index.ts` (`isAuthenticated`, `getCachedUser`, `clearAuth`)
**Args:** `[action?]` (optional: `status` \| `logout`)
**Options:** None

### Behavior

**`status` (default):** Shows current auth status. If authenticated: shows user name and login. If not: shows "Not authenticated" with hint.

**`logout`:** Clears cached token from `~/.100x/auth.json`.

### What It Renders

**Authenticated:**
```
  Authenticated as: Aryan Batra (aryanbatras)
```

**Not authenticated:**
```
  Not authenticated.
  Run 100x submit to authenticate.
```

### Testing

```bash
# Show status (default)
100x auth

# Show status explicitly
100x auth status

# Logout
100x auth logout
```

**Check:**
- ✅ Authenticated state shows user details
- ✅ Not-authenticated state shows hint
- ✅ Logout clears auth (verify with `100x auth` after)

---

## 18. `100x logout`

**File:** `cli/src/commands/logout.tsx`
**Auth:** `auth/index.ts` (`clearAuth`)
**Args:** None | **Options:** None

### Behavior

Clears the cached auth token. Simple alias for `100x auth logout` (without args).

### What It Renders

```
  Authentication cleared.
  Run 100x login to authenticate again.
```

### Testing

```bash
100x logout
```

**Check:**
- ✅ Auth cleared
- ✅ Shows "Authentication cleared" message
- ✅ Verify: `100x auth` shows "Not authenticated"

---

## 19. `100x update`

**File:** `cli/src/commands/update.tsx`
**Actions:** None (self-contained — uses `execaSync`)
**Args:** None | **Options:** None

### Behavior

1. Reads local version from its own `package.json`
2. Calls `npm view @100xsystems/cli version` to check npm registry
3. Compares versions. If same: up to date. If different: shows update command.

### What It Renders

**Up to date:**
```
  ✓ You're up to date!
  Version: 0.1.0
```

**Update available:**
```
  ⟳ Update available!

  Current: 0.1.0
  Latest:  0.2.0

  npm update -g @100xsystems/cli → upgrade
```

### Testing

```bash
100x update
```

**Check:**
- ✅ Up-to-date message when local matches npm
- ✅ Update message when versions differ
- ✅ Shows current and latest versions
- ✅ Shows upgrade command
- ✅ Graceful failure if no network

---

## 20. Sub-Commands: `contribute track` & `contribute lesson`

These extend the `100x contribute` command. See [Section 13](#13-100x-contribute--curriculum-contribution-new) for full details.

### `100x contribute track`

**Args:** `track`, `[systemSlug]`, `[language]` (e.g., `java`, `python`, `rust`)
**Purpose:** Adds a new language track to an existing system.

```bash
# Add Java track to Claude Code
100x contribute track claude-code java

# With custom title
100x contribute track microservices go --title "Go Microservices"
```

**Creates:**
- `curriculum/systems/[slug]/track-[lang]/`
- `curriculum/systems/[slug]/track-[lang]/module-1-introduction/`
- `curriculum/systems/[slug]/track-[lang]/module-1-introduction/01-lesson-introduction.md`
- Updates `index.md` with new track in frontmatter

### `100x contribute lesson`

**Args:** `lesson`, `[systemSlug]`
**Options:** `--language`, `--title`, `--description`
**Purpose:** Adds a new module + lesson to an existing track.

```bash
# Add lesson to typescript track
100x contribute lesson claude-code --language typescript --title "Docker Deployment"

# Add lesson with custom module name
100x contribute lesson microservices --language java "Advanced Topics"
```

**Creates:**
- `curriculum/systems/[slug]/track-[lang]/module-[N]-[title]/`
- `curriculum/systems/[slug]/track-[lang]/module-[N]-[title]/[NN]-lesson-[slug].md`

---

## Edge Case Testing Matrix

| Command | Edge Case | Expected Behavior |
|---------|-----------|-------------------|
| `list` | Non-existent slug | "System not found" |
| `list` | Empty curriculum | "No systems found" |
| `init` | Already exists dir | "Directory already exists" |
| `init` | Non-existent system | "System not found" |
| `validate` | No `.100x.json` | "Run `100x init` first" |
| `verify` | No `.100x.json` | "Run `100x init` first" |
| `verify` | No spec checks | Only doc + structure checks |
| `review` | No API key | Clear error with setup instructions |
| `review` | Bad API key | HTTP error shown |
| `review` | Local model offline | Connection refused error |
| `submit` | No `.100x.json` | "Run `100x init` first" |
| `submit` | Cancelled at confirm | "Submission cancelled." |
| `submit` | PR creation fails | Manual git instructions shown |
| `quiz` | Non-existent system | "System not found" |
| `quiz` | No quizzes | "No quizzes found" |
| `challenge` | Invalid `--start` slug | "Challenge not found" |
| `challenge` | No challenges | "No challenges found" |
| `progress` | No progress file | Started fresh (empty progress) |
| `learn` | No systems with tracks | All systems skipped |
| `public init` | Already configured | Overwritten with new gist |
| `public update` | Never initialized | "No gist configured" |
| `contribute init` | Existing system | "System already exists" |
| `contribute track` | Non-existent system | "System not found" |
| `contribute track` | Duplicate track | "Track already exists" |
| `contribute` | Outside monorepo | "Curriculum directory not found" |
| `doctor` | Tool not installed | "not found (optional/required)" |
| `doctor` | Non-existent system slug | Falls back to all tools |
| `resources` | No resources | "No curated resources yet" |
| `login` | Already authenticated | Shows user info |
| `login --force` | Already authenticated | Re-authenticates |
| `auth logout` | Not authenticated | Still shows "Auth cleared" |
| `update` | No network | "Could not check for updates" |

---

## Quick Smoke Test Checklist

```bash
# ─── Quick Verification ───

cd /Users/aryanbatra/IdeaProjects/100xsystems

# 1. Build passes
cd cli && npm run build && echo "BUILD OK" || echo "BUILD FAILED"

# 2. TypeScript compiles cleanly
npx tsc --noEmit && echo "TSC OK" || echo "TSC FAILED"

# 3. Core commands work
100x list                     # Shows systems
100x doctor                   # Shows environment
100x learn                    # Shows dashboard
100x public status            # Shows build-in-public status

# 4. Scaffolding works
100x init claude-code --output /tmp/100x-smoke
cd /tmp/100x-smoke
100x validate                  # Shows validation results
100x verify                    # Shows spec + doc + structure checks
cd -

# 5. Contribution scaffold works
100x contribute init smoke-test --description "Smoke test"
rm -rf curriculum/systems/smoke-test/

# 6. Clean up
rm -rf /tmp/100x-smoke

echo "ALL SMOKE TESTS PASSED"
```
