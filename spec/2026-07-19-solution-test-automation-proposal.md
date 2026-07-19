# Automated Solution/Test Sync & Verification System

> **Date:** July 19, 2026
> **Status:** Final proposal — ready for implementation
> **Based on research:** Exercism, The Odin Project, Codecademy, freeCodeCamp, Frontend Masters, GitHub Classroom

---

## 1. Executive Summary

### The Problem

Every 100xSystems lesson has:
- A `lesson.md` with the content
- A `tests/` directory with behavioral tests
- An optional `solution/` directory with expected implementation code

These three artifacts must stay in sync. Currently, they are maintained **manually**. When a test changes, the solution must be hand-updated. When a new lesson is added, solutions must be hand-crafted. This creates **4 categories of drift** (detailed below) that silently break the user experience.

### The Solution

A **3-layer automation system** inspired by Exercism's `configlet` toolchain and GitHub Classroom's CI patterns:

| Layer | Tool | What it does |
|-------|------|-------------|
| **1. Manifest** | `100xsystems audit-solutions` | Scans test files and generates a `.solution-manifest.json` documenting what each test expects |
| **2. Verify** | `100xsystems verify-solutions` | Cumulatively copies solutions, compiles, runs tests, reports pass/fail per lesson |
| **3. CI** | `.github/workflows/verify-solutions.yml` | Runs `audit-solutions` + `verify-solutions` on every push to any system repo |

---

## 2. Research-Informed Design Decisions

### Decision 1: Manifest-Based Verification (Exercism Model)

**Exercism** uses a canonical `problem-specifications` repository + `configlet sync` to synchronize tests across 70+ language tracks. Each exercise has a `canonical-data.json` with all test cases, and each track has a `tests.toml` mapping which tests are implemented.

**Our adaptation:** Instead of a separate `problem-specifications` repo, we auto-generate a `.solution-manifest.json` directly from the test files. This manifest documents:
- Each file the test checks for (`fileExists`, `dirExists`)
- Each regex pattern the test checks against (`fileMatches`)
- Each module the test imports (`importModule`)
- Build requirements (`expectBuildSucceeds`)

**Why manifest over canonical repo:**
- Less overhead (no third repo to maintain)
- Auto-generated, never manually edited
- Committed alongside the tests (visible in PR diffs)
- Any contributor can see what changed without reading raw test files

### Decision 2: Cumulative Verification (Progressive Build Model)

Unlike Exercism's isolated per-exercise testing, 100xSystems lessons **build on each other**. Lesson 5's test imports code written in lesson 3. Testing in isolation would miss cross-lesson import errors.

**Algorithm:**

```
For each lesson L1 through Ln:
  1. Create temp project T
  2. Scaffold:            npm init + tsconfig + package.json
  3. For each lesson 1..i:
     Copy solution/Li into T  (newer files overwrite older)
  4. Run: npm run build       → check compilation
  5. Run: vitest run          → check tests pass
  6. Record: { lesson, compiles, testsPassed/totalTests }
```

**This catches:**
- **Category A:** Test expects file X but solution has file Y — test fails
- **Category B:** Lesson 5's solution removes a file lesson 3 depends on — build fails
- **Category C:** New tsconfig strictness breaks existing solutions — build fails

### Decision 3: Git-Based CI Triggers (GitHub Classroom Model)

GitHub Classroom uses template repositories + CI workflows triggered on every push. We adopt the same model:

- Each system repo (`claude-code`, `microservices`, etc.) has its own `.github/workflows/verify-solutions.yml`
- CI triggers on push to `main` and on every PR
- CI runs `configlet lint`-style structural checks + `configlet sync`-style verification

**Why per-repo rather than centralized:**
- Each system repo is independently deployable
- CI results are visible in the repo's own PR checks
- Contributors see validation results without leaving the repo

### Decision 4: No In-Browser Execution (Unlike Codecademy/freeCodeCamp)

Codecademy and freeCodeCamp run tests in the browser or in ephemeral containers. We deliberately **do not** do this because:

- 100xSystems is a **local-first** platform (students work in their own IDE)
- The CLI already handles local test execution via the `validate` command
- Browser execution would require a complex sandbox infrastructure (Codecademy spends millions on this)
- Our test runner executor already handles isolated temp directory execution

---

## 3. The Gap Analysis (Reaffirmed by Research)

| Category | Risk | Exercism | Odin Project | Codecademy | Our Approach |
|----------|------|----------|--------------|------------|--------------|
| **A: Test ↔ Solution mismatch** | 🔴 High | Catches via `canonical-data.json` + `configlet sync` | No detection | Catches via test runner in CI | **Manifest comparison + cumulative verify** |
| **B: Cross-lesson dependency** | 🔴 High | N/A (exercises are independent) | N/A (project-based) | N/A (independent exercises) | **Cumulative copy + build + test** |
| **C: Config/environment drift** | 🟡 Medium | Catches via `configlet lint` | No detection | Internal CI | **TypeScript compile check in verify** |
| **D: Manual process gaps** | 🟢 Low | Catches via `configlet fmt` + PR review | Community-driven | Internal tooling | **Auto-generated manifest + CI gate** |

### Key Insight from Research

**Exercism's biggest contribution** is the `tests.toml` UUID-based tracking. Each canonical test has a UUID. When a track chooses to implement or skip a test, it records the UUID in `tests.toml`. This means:
1. You can see exactly which canonical tests are implemented
2. When canonical tests are added/removed, `configlet sync` detects the drift
3. CI fails if `tests.toml` is out of sync with the canonical data

**Our equivalent:** The `.solution-manifest.json` acts as our `tests.toml`. Instead of UUIDs, it records file paths and regex patterns. When a test changes, the manifest is regenerated, and CI compares the old manifest's expectations against the current solution files.

---

## 4. Detailed Implementation Design

### 4.1 The Solution Manifest (`audit-solutions`)

#### Command

```
100xsystems audit-solutions <system> [track]
```

#### Input

Scans the system repo's track directories for test files.

#### Output

Writes `.solution-manifest.json` at the track root:

```json
{
  "version": 2,
  "generatedAt": "2026-07-19T12:00:00Z",
  "system": "claude-code",
  "track": "track-typescript",
  "lessons": {
    "lesson-intro-and-setup": {
      "slug": "lesson-intro-and-setup",
      "title": "Introduction & Project Setup",
      "order": 1,
      "testFile": "tests/behavior.test.ts",
      "expectedFiles": [
        { "path": "src/cli.ts", "checks": ["exists", "matches:/Command|program/"] },
        { "path": "src/index.ts", "checks": ["exists", "matches:/createCLI|main/"] },
        { "path": "src/agent/", "checks": ["dirExists"] },
        { "path": "src/tools/", "checks": ["dirExists"] },
        { "path": "src/llm/", "checks": ["dirExists"] }
      ],
      "expectedTests": 6,
      "hasBuildCheck": true
    },
    "lesson-build-cli-tool": {
      ...
    }
  },
  "files": {
    "src/cli.ts": ["lesson-intro-and-setup", "lesson-build-cli-tool"],
    "src/index.ts": ["lesson-intro-and-setup", "lesson-build-cli-tool"],
    "src/tools/registry.ts": ["lesson-agent-loop", "lesson-implement-tools"],
    ...
  }
}
```

The `files` map at the bottom is key — it tracks which **multiple** lessons provide versions of the same file. This is critical for cumulative verification.

#### Implementation: `parseTestExpectations()`

```typescript
function parseTestExpectations(testContent: string): ExpectedFile[] {
  const expectations: ExpectedFile[] = [];
  
  // fileExists('path')
  const fileExistsRegex = /fileExists\(['"]([^'"]+)['"]\)/g;
  let match;
  while ((match = fileExistsRegex.exec(testContent)) !== null) {
    expectations.push({ path: match[1], checks: ['exists'] });
  }
  
  // dirExists('path')
  const dirExistsRegex = /dirExists\(['"]([^'"]+)['"]\)/g;
  while ((match = dirExistsRegex.exec(testContent)) !== null) {
    expectations.push({ path: match[1], checks: ['dirExists'] });
  }
  
  // fileMatches('path', /pattern/)
  const fileMatchesRegex = /fileMatches\(['"]([^'"]+)['"],\s*(['/].+?['/]\w*)\)/g;
  while ((match = fileMatchesRegex.exec(testContent)) !== null) {
    expectations.push({ path: match[1], checks: [`matches:${match[2]}`] });
  }
  
  // fileContains('path', 'text')
  const fileContainsRegex = /fileContains\(['"]([^'"]+)['"],\s*['"](.+?)['"]\)/g;
  while ((match = fileContainsRegex.exec(testContent)) !== null) {
    expectations.push({ path: match[1], checks: [`contains:${match[2]}`] });
  }
  
  // importModule('path')
  const importModuleRegex = /importModule\(['"]([^'"]+)['"]\)/g;
  while ((match = importModuleRegex.exec(testContent)) !== null) {
    expectations.push({ path: `dist/${match[1]}`, checks: ['importable'] });
  }
  
  // expectBuildSucceeds()
  if (/expectBuildSucceeds/.test(testContent)) {
    // stored as hasBuildCheck
  }
  
  return expectations;
}
```

#### CLI Command File

```typescript
// cli/src/commands/audit-solutions.tsx
// Follows the same pattern as solution.tsx:
// 1. Read config or accept system/track args
// 2. Generate manifest
// 3. Write to track root
// 4. Report summary
```

### 4.2 The Solution Verifier (`verify-solutions`)

#### Command

```
100xsystems verify-solutions <system> [track]
```

#### Algorithm (Detailed)

```typescript
export async function verifySolutions(systemSlug: string, trackSlug: string): Promise<VerificationReport> {
  const lessons = getTrackFlatLessons(systemSlug, trackSlug);
  const results: LessonVerification[] = [];
  
  // Shared temp dir that accumulates solutions
  const tmpDir = await createTempDir('verify-solutions');
  const cumulativeFiles = new Map<string, string>(); // filePath → originLesson
  
  try {
    // Scaffold a fresh project
    await scaffoldProject({
      targetDir: tmpDir,
      systemSlug,
      systemTitle: slugToDisplayName(systemSlug),
      trackSlug,
      language: trackSlug.replace('track-', ''),
    });
    
    // Install deps
    await execa('npm', ['install'], { cwd: tmpDir, timeout: 120_000 });
    
    for (const lesson of lessons) {
      const solutionDir = path.join(SYSTEMS_DIR(), systemSlug, trackSlug, findLessonDir(lesson.slug), 'solution');
      
      // Copy cumulative solution files
      if (fs.existsSync(solutionDir)) {
        copyCumulative(solutionDir, tmpDir, cumulativeFiles, lesson.slug);
      }
      
      // Copy test file
      const testFile = path.join(SYSTEMS_DIR(), systemSlug, trackSlug, findLessonDir(lesson.slug), 'tests', 'behavior.test.ts');
      if (fs.existsSync(testFile)) {
        fs.copyFileSync(testFile, path.join(tmpDir, 'test.spec.ts'));
      }
      
      // Step A: TypeScript compilation check
      const tscResult = execa.sync('npx', ['tsc', '--noEmit'], {
        cwd: tmpDir, timeout: 30_000, reject: false,
      });
      
      // Step B: Run behavioral tests
      let testResult: TestResult = { total: 0, passed: 0, failed: 0 };
      if (hasTests(lesson)) {
        const vitestResult = execa.sync('npx', ['vitest', 'run', '--reporter', 'json'], {
          cwd: tmpDir, timeout: 60_000, reject: false,
        });
        testResult = parseVitestJson(vitestResult.stdout);
      }
      
      results.push({
        lessonSlug: lesson.slug,
        lessonTitle: lesson.title,
        compiles: tscResult.exitCode === 0,
        compileErrors: tscResult.stderr,
        tests: testResult,
        filesProvided: cumulativeFiles.size,
      });
    }
  } finally {
    fs.rmSync(tmpDir, { recursive: true, force: true });
  }
  
  return { system: systemSlug, track: trackSlug, results, timestamp: new Date().toISOString() };
}
```

#### Cumulative File Tracking

```typescript
function copyCumulative(
  solutionDir: string, 
  targetDir: string, 
  fileOrigin: Map<string, string>, 
  lessonSlug: string
): void {
  const files = collectFiles(solutionDir);
  
  for (const [relPath, content] of files) {
    const targetPath = path.join(targetDir, relPath);
    const prevOrigin = fileOrigin.get(relPath);
    
    // Track which lesson most recently provided this file
    fileOrigin.set(relPath, lessonSlug);
    
    // Write the file (newer solution = more complete version)
    fs.mkdirSync(path.dirname(targetPath), { recursive: true });
    fs.writeFileSync(targetPath, content);
  }
}
```

#### Edge Cases

| Scenario | Behavior |
|----------|----------|
| Lesson has no solution dir | Skip copy, still run tests against cumulative code |
| Lesson has no test file | Skip test run, still check compilation |
| Solution file is empty | Copy empty file — test will fail naturally |
| Solution introduces new file not in earlier lessons | File is added to cumulative set |
| Later solution removes a file earlier one created | File stays unless overwritten by name |
| npm install fails | Fail verification with clear error message |
| vitest timeout | Record as test failure with timeout detail |

### 4.3 CI Workflow

#### File: `.github/workflows/verify-solutions.yml`

```yaml
name: Verify Solutions
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  audit:
    name: Audit solutions
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Generate solution manifest
        run: npx 100xsystems audit-solutions ${{ github.event.repository.name }} track-typescript
        continue-on-error: true
      
      - name: Check manifest changes
        run: |
          if ! git diff --exit-code .solution-manifest.json; then
            echo "⚠️  Solution manifest updated. Commit the new manifest file."
          fi
  
  verify-typescript:
    name: Verify TypeScript track
    runs-on: ubuntu-latest
    needs: audit
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Verify all solutions
        run: npx 100xsystems verify-solutions ${{ github.event.repository.name }} track-typescript
      
      - name: Upload verification report
        uses: actions/upload-artifact@v4
        with:
          name: verification-report
          path: .verification-report.json
  
  verify-java:
    name: Verify Java track
    if: hashFiles('track-java/**/*') != ''
    runs-on: ubuntu-latest
    needs: audit
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '17'
      
      - name: Verify all solutions
        run: npx 100xsystems verify-solutions ${{ github.event.repository.name }} track-java
```

#### Badge in README

```markdown
![Verify Solutions](https://github.com/100xsystems/claude-code/actions/workflows/verify-solutions.yml/badge.svg)
```

---

## 5. File Manifest Tracking

### The "Multi-Owner" Problem

Many files are owned by multiple lessons:

```
src/cli.ts           → Lessons 1, 2
src/tools/registry.ts → Lessons 3, 4
src/index.ts         → Lessons 1, 2
```

When verifying cumulatively, if lesson 4's solution has `src/tools/registry.ts` but accidentally removes the `readFile` tool that lesson 3's test expects, the verifier catches it because:
1. Lesson 3's test passes when verified with lesson 3's solution
2. But when lesson 4's solution is copied (overwriting registry.ts), lesson 3's test context changes
3. The verifier runs ALL tests against the cumulative state, catching the regression

### Solution File Origin Tracking

The manifest tracks which lesson "owns" each file:

```json
"files": {
  "src/cli.ts": { "currentOwner": "lesson-build-cli-tool", "previousOwners": ["lesson-intro-and-setup"] },
  "src/index.ts": { "currentOwner": "lesson-build-cli-tool", "previousOwners": ["lesson-intro-and-setup"] },
  "src/agent/loop.ts": { "currentOwner": "lesson-agent-loop", "previousOwners": [] },
  "src/tools/registry.ts": { "currentOwner": "lesson-implement-tools", "previousOwners": ["lesson-agent-loop"] }
}
```

This enables the CI to detect:
- **Orphaned files:** A file that was in the solution but is no longer needed (flagged as warning)
- **Unintentional overwrites:** A newer solution overwrites a file but removes critical functionality

---

## 6. Reporting & Visualization

### CLI Output

```
$ 100xsystems verify-solutions claude-code track-typescript

📋 Solution Verification Report
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
System: claude-code  |  Track: track-typescript
Timestamp: 2026-07-19T14:30:00Z

  Lesson                    Compile  Tests  Status
  ────────────────────────────────────────────────
  1  Intro & Setup          ✅      6/6    ✅ PASS
  2  Build CLI Tool         ✅      6/6    ✅ PASS
  3  Agent Loop             ✅      6/6    ✅ PASS
  4  Implement Tools        ✅      6/6    ✅ PASS
  5  CLI Foundations Quiz   ⏭️      —      ⏭️ No tests
  6  CLI Challenge          ✅      —      ✅ Build OK
  7  LLM Integration        ✅      5/5    ✅ PASS
  8  Context Management     ✅      3/3    ✅ PASS
  9  Tool/Plugin System     ✅      3/3    ✅ PASS

  ────────────────────────────────────────────────
  Total:  9 lessons · 8 passed · 0 failed · 1 skipped
```

### Machine-Readable Output

`--json` flag produces a structured JSON report for CI consumption:

```json
{
  "system": "claude-code",
  "track": "track-typescript",
  "passed": 8,
  "failed": 0,
  "skipped": 1,
  "results": [
    {
      "lesson": "lesson-intro-and-setup",
      "title": "Introduction & Project Setup",
      "compiles": true,
      "tests": { "total": 6, "passed": 6, "failed": 0 },
      "status": "pass"
    }
  ],
  "duration": 185000,
  "timestamp": "2026-07-19T14:30:00Z"
}
```

---

## 7. Implementation Roadmap

### Phase 1 (Week 1): Core Infrastructure

| Task | Files | Est. |
|------|-------|------|
| `SolutionManifest` type + generator | `cli/src/actions/audit-solutions.ts` | 4h |
| `parseTestExpectations()` — extract test expectations from test files | `cli/src/actions/audit-solutions.ts` | 4h |
| `100xsystems audit-solutions` command | `cli/src/commands/audit-solutions.tsx` | 2h |
| Typecheck + build + unit test | — | 1h |
| Push CLI changes to repo | — | — |

### Phase 2 (Week 2): Solution Verification

| Task | Files | Est. |
|------|-------|------|
| `verifySolutions()` — cumulative copy + typecheck + test runner | `cli/src/actions/verify-solutions.ts` | 8h |
| Cumulative file tracking + origin logger | `cli/src/actions/verify-solutions.ts` | 2h |
| `100xsystems verify-solutions` command | `cli/src/commands/verify-solutions.tsx` | 2h |
| JSON reporter for CI consumption | `cli/src/actions/verify-solutions.ts` | 1h |
| Typecheck + build + integration test | — | 2h |

### Phase 3 (Week 3): CI Integration

| Task | Files | Est. |
|------|-------|------|
| Create `.github/workflows/verify-solutions.yml` in claude-code | `claude-code/.github/workflows/verify-solutions.yml` | 2h |
| Same for microservices | `microservices/.github/workflows/verify-solutions.yml` | 1h |
| Badge in README | `claude-code/README.md` | 0.5h |
| Test CI on real PR | — | 1h |

### Phase 4 (Week 4 — Stretch): Solution Generation

| Task | Files | Est. |
|------|-------|------|
| `generateSolution()` — auto-create solution stubs from manifest | `cli/src/actions/generate-solution.ts` | 6h |
| `100xsystems generate-solution <lesson>` command | `cli/src/commands/generate-solution.tsx` | 1h |
| Smart merge of new solution with existing cumulative state | `cli/src/actions/generate-solution.ts` | 3h |

### Total Estimate: ~40 hours

---

## 8. What This Unlocks

| Capability | Before | After |
|-----------|--------|-------|
| Solution drift detection | Manual discovery | Auto-detected on every commit |
| Cross-lesson regression | Impossible to detect | Caught by cumulative verify |
| New contributor confidence | "Hope I didn't break anything" | CI says ✅ or ❌ |
| Solution creation speed | Hand-craft every file | Auto-generate stubs from tests |
| Onboarding new systems | Manual audit | Automated verification |
| Curriculum quality score | Unknown | Dashboard per system/track |

---

## 9. Comparison: 100xSystems vs Other Platforms

| Feature | Exercism | TOP | Codecademy | **100xSystems (Proposed)** |
|---------|----------|-----|------------|--------------------------|
| **Test/Solution sync** | `configlet sync` + canonical data | Manual | Internal CI | Manifest + cumulative verify |
| **Verification type** | Per-exercise isolated | Manual/peer review | Per-exercise | **Cumulative (cross-lesson)** |
| **CI integration** | `configlet lint` + `configlet fmt` | None | Internal | Manifest check + verify |
| **Solution format** | Single file per exercise | Any | Any | **Progressive files across lessons** |
| **Drift detection** | `canonical-data.json` diff | None | Internal | `.solution-manifest.json` diff |
| **Auto-generation** | Test generators (limited) | None | None | Planned (Phase 4) |

---

## 10. Risks & Mitigations

| Risk | Likelihood | Mitigation |
|------|-----------|------------|
| Manifest fails to parse complex test files | Medium | Fall back to running tests; manifest is optimization, not requirement |
| Cumulative verification is slow (9 lessons × 60s = 9min) | Medium | Parallelize per-lesson; cache npm install; use --reporter compact |
| False positives from env differences (CI vs local) | Low | Use same Node version in CI; pin dependencies |
| Solutions become stale after refactors | Low | CI fails on drift; contributor PR fixes |
| Multiple tracks (TypeScript + Java) need different runners | Medium | Verify runner dispatches by track type (vitest vs maven vs cargo) |

---

*Generated: July 19, 2026 — Based on research of Exercism, The Odin Project, Codecademy, freeCodeCamp, Frontend Masters, and GitHub Classroom patterns.*
