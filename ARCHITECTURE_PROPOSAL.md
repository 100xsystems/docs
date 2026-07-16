# 100xSystems — Complete Architectural Proposal

> **Date:** July 15, 2026
> **Status:** Implementation Roadmap
> **Author:** Synthesized from codebase audit + conversation analysis + external research (Exercism, CodeCrafters)

---

## Table of Contents

1. [Current State Analysis](#1-current-state-analysis)
2. [Critical Gaps](#2-critical-gaps)
3. [The Architectural Shift](#3-the-architectural-shift)
4. [Validator Plugin Architecture](#4-validator-plugin-architecture)
5. [GitHub PR Submission Flow](#5-github-pr-submission-flow)
6. [Review Workflow](#6-review-workflow)
7. [Certificate System](#7-certificate-system)
8. [Implementation Roadmap](#8-implementation-roadmap)
9. [Solved Hard Problems](#9-solved-hard-problems)

---

## 1. Current State Analysis

### CLI (`cli/`)

**Framework:** Pastel (Ink-based React CLI framework)
**Commands:** `auth`, `init`, `list`, `quiz`, `submit`, `validate`, `verify`, `doctor`, `resources`, `progress`, `challenge`, `update`, `search`, `logout`
**Auth:** GitHub PKCE OAuth + Device Flow fallback
**UI:** Interactive dashboard with tabs (BUILD, QUIZ, DOCTOR, RESOURCES)
**Readers:** system-reader, quiz-reader, spec-reader, resource-reader (all read from `curriculum/`)
**Scaffold:** Creates `.100x.json` + review package (README, design/decisions.md, design/architecture.md, design/tradeoffs.md, verification/checklist.md)
**Validation:** Checks documentation completeness, structure, git history (HARDCODED — no plugin system)
**Submit:** Packages review package into `submissions/` folder (NO PR automation)
**Progress:** Local `~/.100x/progress.json` (NO sync with website)

### Website (`website/`)

**Framework:** Next.js 15 App Router (SSG at build time)
**Pages:** Home, Systems, Principles, Patterns, Tools, Technologies, Languages, Search
**Components:** Full design system with tokens → atoms → composites → layouts → animations
**Curriculum Bridge:** `mdx.ts` reads from `curriculum/` at build time
**Reading Experience:** Custom reading controls (font, size, line height, mode, code theme, width)

### Curriculum (`curriculum/`)

```
curriculum/
  systems/
    {slug}/
      index.md          — metadata (title, description, difficulty, tags)
      architecture/     — architecture docs
      diagrams/         — diagram files
      tradeoffs/        — trade-off analysis
      specification/    — system specs
      implementation/   — implementation guides
      quizzes/          — quiz questions
      challenges/       — challenge prompts
      resources/        — curated resource links
  knowledge-base/
    principles/         — CAP, SOLID, ACID, etc.
    patterns/           — Factory, Singleton, Observer, etc.
    tools/              — Docker, Kubernetes, Terraform, etc.
    technologies/       — Kafka, Redis, PostgreSQL, etc.
  search/               — JSON files with curated resource links
  languages/            — (empty/draft)
```

**Problem:** Content is organized by **content type** (architecture/, tradeoffs/) instead of **learning sequence** (modules/lessons/). Most content is dummy/empty.

---

## 2. Critical Gaps

| # | Gap | Severity | Details |
|---|-----|----------|---------|
| 1 | **Folder-based vs lesson-based** | 🔴 CRITICAL | `architecture/`, `tradeoffs/`, `implementation/` are content types, not learning sequences |
| 2 | **No language tracks** | 🔴 CRITICAL | Java vs TypeScript vs Rust of same system need different content — no support |
| 3 | **No validator plugins** | 🔴 CRITICAL | Validation is hardcoded to check doc existence — can't verify actual implementations |
| 4 | **No PR submission** | 🟡 HIGH | `submit` packages files locally — doesn't fork/commit/push/create PR |
| 5 | **No review system** | 🟡 HIGH | No PR-based review pipeline, no reviewer queue, no status tracking |
| 6 | **Dummy content** | 🔴 CRITICAL | Near-zero real lesson content — just folder structures and templates |
| 7 | **No website↔CLI sync** | 🟡 MEDIUM | Progress is local-only, website has no user state |
| 8 | **No certificates** | 🟢 LOW | Discussed but not implemented |
| 9 | **No community submissions view** | 🟡 MEDIUM | Website doesn't show verified community implementations |
| 10 | **No knowledge base cross-linking** | 🟡 MEDIUM | Lessons don't reference KB entries |
| 11 | **No AI review** | 🟢 LOW | `review` command is a concept, not built |

---

## 3. The Architectural Shift

### Principle: Curriculum must be organized around the learner's journey, not the author's folder structure.

### FROM (Current):

```
systems/claude-code/
  index.md
  architecture/
    01-architecture-overview.md
    09-agent-loop.md
  tradeoffs/
    01-tradeoffs.md
  specification/
    01-system-spec.md
  implementation/
    04-implementation-guide.md
  quizzes/
    01-quiz.md
  challenges/
    01-challenge.md
```

### TO (Target):

```
systems/claude-code/
  index.md                       → metadata + track definitions
  track-typescript/               → TypeScript-specific lessons
    module-1-cli-foundations/
      01-lesson-intro.md
      02-lesson-setup.md
      03-lesson-build-cli.md
      04-lesson-test.md
      quiz.md
      challenge.md
      validator/
        http.js                   → validator for this lesson
    module-2-ai-integration/
      ...
  track-java/
    module-1-cli-foundations/
      ... (different content, different code)
  resources/                     → shared across tracks
    papers.md
    videos.md
```

### Lesson Frontmatter (Single Source of Truth):

```markdown
---
title: "Build the CLI Tool"
order: 3
module: "CLI Foundations"
track: "typescript"
difficulty: "Beginner"
estimated_time: "45 min"
prerequisites:
  - "tools/nodejs"
  - "languages/typescript-basics"
knowledge_refs:
  - "principles/single-responsibility"
  - "patterns/factory"
validation:
  - type: file-exists
    path: "src/cli.ts"
  - type: npm-test
    command: "npm test"
  - type: http
    url: "http://localhost:3000/health"
    method: GET
    expect_status: 200
---

# Build the CLI Tool

[Lesson content with explanation, code snippets, architecture, trade-offs, all integrated]
```

**Key insight:** No separate metadata files. No JSON configs. **Everything is in the markdown frontmatter.** Rename a lesson? Move it? The metadata moves with it automatically.

---

## 4. Validator Plugin Architecture

### The Hardest Problem Solved

The CLI should NEVER know how to test specific systems. It should be a **runtime** for lesson-defined validators.

### Architecture

```
cli/src/executors/
  registry.ts             → Validator type registry (maps type → executor)
  types.ts                → ValidationContext, ValidationResult types
  file-exists.executor.ts → Checks if files exist
  http.executor.ts        → Sends HTTP requests, checks responses  
  docker.executor.ts      → Builds images, runs containers
  regex.executor.ts       → Searches code for patterns
  npm-test.executor.ts    → Runs npm test
  junit.executor.ts       → Runs mvn test, parses JUnit XML
  terraform.executor.ts   → Runs terraform validate
  custom.executor.ts      → Executes user-provided JS/TS validator
  cli-command.executor.ts → Runs arbitrary CLI commands, checks exit code
  compose.executor.ts     → Validates docker compose services
```

### Lesson Specifies Validators

In lesson frontmatter:

```yaml
validation:
  - type: http
    request:
      url: "http://localhost:3000/api/health"
      method: GET
    expect:
      status: 200
      body_contains: "\"status\": \"ok\""

  - type: file-exists
    path: "src/main/java/com/example/UserService.java"

  - type: npm-test
    command: "npm run test:integration"
    timeout: 60000

  - type: file-contains
    path: "design/decisions.md"
    contains: "Kafka"
```

### Executor Interface

```typescript
// cli/src/executors/types.ts
export interface ExecutorContext {
  projectDir: string;
  lessonDir: string;
  workspace: string;
}

export interface ExecutorResult {
  status: 'pass' | 'warn' | 'fail';
  message: string;
  details?: string;
}

export interface Executor {
  type: string;
  execute(params: Record<string, any>, ctx: ExecutorContext): Promise<ExecutorResult>;
}
```

### How Validation Runs

```
100x validate
  → Read .100x.json to find current system + track
  → Walk all lessons in the system's track
  → For each lesson:
    → Read frontmatter validation config
    → Execute each validator
    → Collect results
  → Display pass/warn/fail by lesson
```

Validators are the **only** thing in the CLI that knows about implementations. Everything else is generic.

---

## 5. GitHub PR Submission Flow

### Current: Manual

```
100x submit
  → Package files into submissions/
  → Print "Create a PR manually"
  → Done
```

### Target: Fully Automated

```
100x submit
  Step 1: Validate (run all lesson validators)        → Fail fast if broken
  Step 2: Authenticate (GitHub OAuth)                  → Already implemented
  Step 3: Build review package                         → Already implemented
  Step 4: Fork 100xsystems/submissions repo            → Octokit API
  Step 5: Clone fork to temp directory                 → Local git
  Step 6: Copy review package files                    → submissions/{system}/{author}-{lang}-{ts}/
  Step 7: Git add + commit                             → "submission({system}): {author}"
  Step 8: Git push to fork                             → Remote
  Step 9: Create PR via GitHub API                     → Octokit pulls.create()
  Step 10: Return PR URL to user                       → "Your PR: https://github.com/..."
  Step 11: Mark progress as "submitted"
```

### What Goes in the PR

```
submissions/
  microservices/
    aryan-typescript-1712345678/
      README.md                    → Project overview
      design/
        decisions.md               → Engineering Decision Log
        architecture.md            → Architecture explanation  
        tradeoffs.md               → Trade-offs analysis
      verification/
        checklist.md               → Self-assessment
      specification/
        SPECIFICATION.md           → System specification
      metadata.json                → { system, author, language, repositoryUrl, ... }
```

The **code never leaves the author's repository**. The PR is a review package + link to source.

### Octokit Integration

```typescript
// cli/src/actions/submit-pr.ts
import { Octokit } from '@octokit/action';
import { execa } from 'execa';
import fs from 'fs';
import path from 'path';

const SUBMISSIONS_REPO = '100xsystems/submissions';

export async function createSubmissionPR(
  token: string,
  user: string,
  reviewDir: string,
  metadata: Record<string, any>
): Promise<string> {
  const octokit = new Octokit({ auth: token });
  const branchName = `submission/${metadata.system}/${user}-${Date.now()}`;

  // Fork the submissions repo
  const fork = await octokit.repos.createFork({
    owner: '100xsystems',
    repo: 'submissions',
  });

  // Clone, copy, commit, push
  const tmpDir = fs.mkdtempSync(path.join(process.cwd(), '.100x-tmp-'));
  await execa('git', ['clone', fork.data.clone_url, tmpDir], { cwd: process.cwd() });
  await execa('git', ['checkout', '-b', branchName], { cwd: tmpDir });
  // Copy review package
  fs.cpSync(reviewDir, path.join(tmpDir, 'submissions', metadata.system, path.basename(reviewDir)), { recursive: true });
  await execa('git', ['add', '.'], { cwd: tmpDir });
  await execa('git', ['commit', '-m', `submission(${metadata.system}): ${user}`], { cwd: tmpDir });
  await execa('git', ['push', 'origin', branchName], { cwd: tmpDir });

  // Create PR
  const pr = await octokit.pulls.create({
    owner: '100xsystems',
    repo: 'submissions',
    title: `[${metadata.system}] Submission by ${user}`,
    head: `${user}:${branchName}`,
    base: 'main',
    body: generatePRBody(metadata),
  });

  return pr.data.html_url;
}
```

---

## 6. Review Workflow

### Principles (from the conversation):

1. **No assignment.** Reviewers pick up PRs they're interested in (like GitHub issues).
2. **Topics over roles.** Reviewers subscribe to topics (language, system name, technology).
3. **PR labels = status.** `needs-review` → `in-review` → `changes-requested` → `verified`.
4. **Discussions on GitHub.** Not on the website. Don't rebuild threaded discussions.

### How it Works:

1. User submits via `100x submit` → PR created on `100xsystems/submissions`
2. PR gets label `needs-review` (automatically)
3. PR appears on the website's "Pending Reviews" section (fetched via GitHub API at build time)
4. A reviewer sees the PR and assigns themselves (comments `/review`)
5. PR label changes to `in-review`
6. Reviewer reads review package + checks linked repository
7. Reviewer leaves comments on the PR
8. If changes needed → label `changes-requested`, author updates documentation
9. If accepted → label `verified`, PR gets merged
10. Certificate is triggered via GitHub Actions

### Website Display:

```
Community Implementations
  ┌─────────────────────────────────────────────┐
  │  Microservices                              │
  │                                             │
  │  ✓ @aryan — TypeScript — PR #42            │
  │    Reviewed by @sarah, @bob                 │
  │    Completed: July 2026                     │
  │                                             │
  │  ✓ @alice — Spring Boot — PR #38            │
  │    Reviewed by @aryan                       │
  │    Completed: June 2026                     │
  └─────────────────────────────────────────────┘
```

Each entry links to the GitHub PR and the author's repository.

---

## 7. Certificate System

### Architecture (Backend-Free):

```
PR merged with label "verified"
  ↓
GitHub Actions triggers
  ↓
Reads metadata.json from merged PR
  ↓
Generates certificate HTML + signs it with a key
  ↓
Commits certificate to certificates/ directory
  ↓
Website rebuilds → certificate page is live
```

### Certificate Verification URL:

```
https://100xsystems.dev/certificates/{sha256-hash}
```

### What the Page Shows:

```
100xSystems — Certificate of Completion

  System:       Microservices
  Author:       @aryan
  Language:     TypeScript
  Repository:   github.com/aryan/microservices
  Review PR:    #42
  Verified By:  @sarah, @bob
  Completed:    July 15, 2026

  [Verify on GitHub] → links to the merged PR
```

### GitHub Actions Workflow:

```yaml
# .github/workflows/certificate.yml
name: issue-certificate
on:
  pull_request:
    types: [closed]
    branches: [main]

jobs:
  certificate:
    if: github.event.pull_request.merged == true && 
        contains(github.event.pull_request.labels.*.name, 'verified')
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Generate Certificate
        run: |
          PR_NUMBER=${{ github.event.pull_request.number }}
          # Find metadata.json
          # Generate certificate.html with PR details
          # Store as certificates/{pr-number}/index.html
      - name: Commit Certificate
        run: |
          git add certificates/
          git commit -m "certificate: PR #${PR_NUMBER}"
          git push
```

---

## 8. Implementation Roadmap

### Priority Order:

| Priority | Feature | Why This Order |
|----------|---------|----------------|
| P0 | **Curriculum lessons** — Restructure into modules/lessons with frontmatter | Everything depends on content structure |
| P0 | **Validator executors** — Implement 5-10 validator types | Automated lesson verification |
| P1 | **GitHub PR submission** — Fork+commit+push+PR in `100x submit` | PR-based review workflow |
| P1 | **Real lesson content** — Claude Code + Microservices | Actual learner value |
| P2 | **Knowledge base cross-linking** — Lessons reference KB | Cohesive learning |
| P2 | **Community submissions on website** — Show verified submissions | Social proof |
| P2 | **Reviewer topic subscriptions** — GitHub labels | Scalable reviews |
| P3 | **Certificate generation** — GitHub Actions | Motivation |
| P3 | **AI review** (`100x review`) — LLM evaluates | Instant feedback |
| P4 | **Progress sync** — CLI ↔ website via API | Cross-device |
| P4 | **Mobile apps** | Platform expansion |

---

## 9. Solved Hard Problems

### Problem 1: How do contributors add new systems without modifying the CLI?

**Solution:** Validator types are in the CLI; lesson frontmatter declares which validators to use. A contributor writes markdown + picks from 10 validator types. The CLI never needs to know about specific systems. New validators can be added independently without changing any command.

### Problem 2: How do we verify implementations across different languages?

**Solution:** Each track (language) has its own lessons. The same `http` validator works for any language — it just sends HTTP requests. Language-specific validators (`npm-test` vs `junit`) verify code correctness per-language. The contract (what the lesson tests) stays the same; the validator implementation varies.

### Problem 3: How do we prevent stale documentation?

**Solution:** The `validate` command already checks `git log` timestamps to warn when code was updated more recently than documentation. Extend this: validators can compare git timestamps of `src/` vs `design/` and warn on mismatch.

### Problem 4: How do we attract and retain volunteer reviewers?

**Solution:** Don't assign. Use GitHub's native PR system. Reviewers self-select based on topics. Reputation builds naturally. Leaderboard shows top reviewers by systems reviewed (not lines of code). PR discussions are visible to all → learning material.

### Problem 5: How do we generate certificates without a backend?

**Solution:** GitHub Actions generates HTML certificates on PR merge. They're served as static pages on the Next.js website. No database, no server. The proof is the merged PR + verifiable certificate URL.

### Problem 6: How do we handle the "triangle" (CLI ↔ Curriculum ↔ Website)?

**Solution:** All three read from the same `curriculum/` directory. The CLI reads it at runtime (file system). The website reads it at build time (Next.js SSG). The curriculum is the single source of truth. No JSON metadata files — frontmatter in each markdown file IS the metadata.

### Problem 7: How do we make the platform "cheat-proof"?

**Solution:** We don't fight AI, we embrace it. The real verification is the **Engineering Decision Log** (design/decisions.md). A person who blindly copies AI output will struggle when asked why they chose a Saga over two-phase commit. The PR review discussion reveals true understanding. Certificates mean "this implementation passed community review" — not "this person never used AI."

---

## Appendix A: File Structure After Implementation

```
100xsystems/
  curriculum/
    systems/
      claude-code/
        index.md
        track-typescript/
          module-1-cli-foundations/
            01-lesson-intro.md
            02-lesson-setup.md
            03-lesson-build-cli.md
            04-lesson-test.md
            quiz.md
            challenge.md
          module-2-ai-integration/
            ...
        track-java/
          module-1-cli-foundations/
            ...
        resources/
          papers.md
          videos.md
      microservices/
        index.md
        track-spring-boot/
          ...
        track-nestjs/
          ...
    knowledge-base/
      principles/
      patterns/
      tools/
      technologies/
    search/
    languages/
  cli/
    src/
      executors/
        registry.ts
        types.ts
        file-exists.executor.ts
        http.executor.ts
        docker.executor.ts
        regex.executor.ts
        npm-test.executor.ts
        junit.executor.ts
        terraform.executor.ts
        custom.executor.ts
        cli-command.executor.ts
        compose.executor.ts
      actions/
        submit.ts
        submit-pr.ts      ← NEW
        validate.ts
        verify.ts
        progress.ts
  website/
    app/
      systems/
        [slug]/
          page.tsx
          [lesson-path]/
            page.tsx       ← NEW sequential lesson reading
  submissions/              ← (or separate repo)
    microservices/
      aryan-typescript-1712345678/
        metadata.json
        ...
```

---

## Appendix B: Key Technology Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| CLI Framework | Pastel (Ink + React) | Already implemented, works well |
| Auth | GitHub PKCE OAuth | Already implemented |
| GitHub API | Octokit | Mature, well-documented |
| Validator runners | Node.js child_process | No Docker needed for simple validators |
| Containerized validators | Docker exec | For complex validators (build, test) |
| Certificate generation | GitHub Actions + static HTML | No backend needed |
| Website framework | Next.js 15 (SSG) | Already implemented |
| Curriculum format | Markdown + frontmatter | Already implemented |
| Content images | curriculum/{system}/images/ | Already considered in mdx.ts |
