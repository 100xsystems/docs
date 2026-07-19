# E2E Solution/Test Sync & Automation Design

> **Date:** July 19, 2026
> **Purpose:** Audit the gaps between `tests/` and `solution/` directories, then design automated verification that catches drift before it reaches users.
> **Status:** Design proposal — ready for implementation.

---

## 1. Current Architecture Audit

### File Layout per Lesson

```
track-typescript/
  01-lesson-intro-and-setup/
    lesson.md          ← Frontmatter: title, order, validation rules
    tests/
      behavior.test.ts ← Test cases using @100xsystems/test-suite-typescript
    solution/          ← Expected implementation code (MANUALLY created)
      src/
        index.ts
        cli.ts
  02-lesson-build-cli-tool/
    lesson.md
    tests/
      behavior.test.ts
    solution/
      src/
        cli.ts         ← Enhanced version of lesson 1's cli.ts
        index.ts
        llm/streaming.ts
  ...
```

### Data Flow for Validation

```
User runs: 100xsystems validate <lesson>

1. Read 100xsystems.json → get { system, track }
2. Resolve curriculum → SYSTEMS_DIR()/{system}/{track}/
3. Find lesson folder by slug
4. Read lesson.md frontmatter → get validation rules
5. For each validator:
   a. file-exists: check file exists in user's project
   b. file-contains: check file matches regex
   c. test-runner: copy user's src/ → temp dir, run vitest
6. Return pass/fail per check
```

### Test Runner Flow (test-runner executor)

```
1. CopyProjectFiles(projectDir → tmpDir)
2. Copy test file (behavior.test.ts) into tmpDir
3. ResolveFileReferences (file: → absolute paths)
4. ResolveLocalTestSuite (find test-suite-typescript package)
5. EnsureDependency (add @100xsystems/test-suite-typescript)
6. EnsureVitestConfig (create vitest.config.ts)
7. InstallDependenciesCached (npm install with caching)
8. Run vitest --reporter json
9. Parse results → ValidationResult[]
```

---

## 2. Gap Analysis: What Can Go Wrong

### Category A: Solution ↔ Test Drift (HIGH PRIORITY)

| Gap | Example | Impact |
|-----|---------|--------|
| **Test expects file A but solution creates file B** | Test: `fileExists('src/llm/tokens.ts')` — Solution has `src/agent/context.ts` instead | ❌ Lesson 8 test fails when solution is applied |
| **Test expects regex pattern not in solution** | Test: `fileMatches('src/cli.ts', /pipe/)` — Solution cli.ts missing `pipe` command | ❌ Test fails silently |
| **Test expects export not in solution** | Test: `importModule('llm/client.js')` expects `LLMClient` class — Solution exports wrong name | ❌ Runtime import error |
| **Solution file path doesn't match test expectation** | Test checks `src/plugins/types.ts` — Solution has `src/tools/plugin-system.ts` | ❌ File not found |

### Category B: Cross-Lesson Dependency Mismatch (HIGH PRIORITY)

| Gap | Example | Impact |
|-----|---------|--------|
| **Lesson 5 test expects code from lessons 1-4** | Test imports from `src/agent/loop.js` but lesson 5 solution only has its own files | ❌ Cumulative build failure |
| **Lesson N solution removes a file lesson N-1 created** | Overwrites `src/registry.ts` with a version that removes a method | ❌ Regression |
| **Solution introduces import from non-existent file** | Lesson 1 solution import `Agent` from `./agent/loop.js` — that file doesn't exist until lesson 3 | ❌ Build fails at lesson 1 |

### Category C: Config/Environment Drift (MEDIUM PRIORITY)

| Gap | Example | Impact |
|-----|---------|--------|
| **tsconfig change not reflected in solution** | Root tsconfig adds `exactOptionalPropertyTypes` — solutions violate it | ❌ Typecheck fails |
| **package.json dep missing from solution** | New lesson needs `zod` — not added to scaffold template | ❌ Build fails |
| **Test-suite helper renamed** | `expectBuildSucceeds()` → `expectBuildPasses()` — all lessons use old name | ❌ All tests fail |

### Category D: Manual Process Gaps (LOW PRIORITY)

| Gap | Impact |
|-----|--------|
| No CI check that solution files compile | Only caught when user runs validate |
| No CI check that tests pass against solutions | Only caught in development |
| No automated generation of solution stubs | Every solution must be hand-crafted |
| No manifest of what each test expects | Must read each test file to understand expectations |

---

## 3. Automation Design

### 3.1 Solution Manifest Generator

**Purpose:** Auto-generate a JSON manifest from test files that documents what the test expects. Compare against solution files to catch drift.

**Command:** `100xsystems audit-solutions`

**Output file:** `track-typescript/.solution-manifest.json`

```json
{
  "version": 1,
  "generatedAt": "2026-07-19T...",
  "lessons": {
    "lesson-intro-and-setup": {
      "slug": "lesson-intro-and-setup",
      "testFile": "tests/behavior.test.ts",
      "expectedFiles": [
        { "path": "src/cli.ts", "pattern": "/Command|program|yargs|meow/i", "type": "fileMatches" },
        { "path": "src/index.ts", "pattern": "/createCLI|main|run|start/i", "type": "fileMatches" },
        { "path": "src/agent/", "type": "dirExists" },
        { "path": "src/tools/", "type": "dirExists" },
        { "path": "src/llm/", "type": "dirExists" }
      ],
      "expectedBuild": true,
      "requiredDeps": ["commander"]
    },
    "lesson-build-cli-tool": { ... }
  }
}
```

**Implementation:**

```typescript
// cli/src/actions/audit-solutions.ts
export async function generateSolutionManifest(systemSlug: string, trackSlug: string): Promise<SolutionManifest> {
  const lessons = getTrackFlatLessons(systemSlug, trackSlug);
  const manifest: SolutionManifest = { version: 1, generatedAt: new Date().toISOString(), lessons: {} };

  for (const lesson of lessons) {
    const testFile = path.join(lesson.dir, 'tests/behavior.test.ts');
    if (!fs.existsSync(testFile)) continue;

    const testContent = fs.readFileSync(testFile, 'utf-8');
    const expectedFiles = parseTestExpectations(testContent);
    
    manifest.lessons[lesson.slug] = {
      slug: lesson.slug,
      testFile: 'tests/behavior.test.ts',
      expectedFiles,
      expectedBuild: testContent.includes('expectBuildSucceeds'),
      requiredDeps: extractRequiredDeps(testContent),
    };
  }

  return manifest;
}
```

### 3.2 Solution Verification Script

**Purpose:** Copy solutions incrementally, run tests, report pass/fail per lesson. Catches ALL gap categories (A-D).

**Script:** `scripts/verify-solutions.sh` (or `100xsystems verify-solutions` command)

**Algorithm:**

```
For each lesson L1 through Ln:
  1. Create temp dir T
  2. Scaffold fresh project in T (100xsystems init)
  3. For each lesson L1 through Li (cumulative):
     a. If solution/Li exists, copy its files into T
  4. Run: npx tsc --noEmit           → check compilation
  5. Run: vitest run tests/behavior   → check tests pass
  6. Record pass/fail for lesson Li
  7. Cleanup T

Report:
  ✅ Lesson 1: Compiles + 6/6 tests pass
  ✅ Lesson 2: Compiles + 6/6 tests pass  
  ❌ Lesson 3: Compiles but 2/6 tests fail
  ⏭️ Lesson 5: No test file
```

**Why cumulative:** Lessons build on each other. Testing lesson 5's solution without lessons 1-4's code would miss the cumulative dependency issue.

**Implementation:**

```typescript
// cli/src/actions/verify-solutions.ts
export async function verifySolutions(systemSlug: string, trackSlug: string): Promise<LessonVerification[]> {
  const lessons = getTrackFlatLessons(systemSlug, trackSlug);
  const results: LessonVerification[] = [];
  const tempDir = await createTempDir('solution-verify');
  const cumulativeFiles = new Map<string, string>(); // path → content

  try {
    for (const lesson of lessons) {
      const solutionDir = path.join(lesson.dir, 'solution');
      const testDir = path.join(lesson.dir, 'tests');
      
      // Copy cumulative solution files into temp project
      if (fs.existsSync(solutionDir)) {
        copyDirContents(solutionDir, tempDir, cumulativeFiles);
      }

      // Copy test file into temp project
      if (fs.existsSync(path.join(testDir, 'behavior.test.ts'))) {
        fs.copyFileSync(
          path.join(testDir, 'behavior.test.ts'),
          path.join(tempDir, 'test.spec.ts')
        );
      }

      // Typecheck
      const tscResult = execa.sync('npx', ['tsc', '--noEmit'], { cwd: tempDir, reject: false });

      // Run tests
      const vitestResult = execa.sync('npx', ['vitest', 'run', '--reporter', 'json'], { cwd: tempDir, reject: false });

      results.push({
        lesson: lesson.slug,
        lessonTitle: lesson.title,
        compiles: tscResult.exitCode === 0,
        testsPassed: parseVitestOutput(vitestResult.stdout),
      });
    }
  } finally {
    fs.rmSync(tempDir, { recursive: true, force: true });
  }

  return results;
}
```

### 3.3 CI Integration

**GitHub Action:** `.github/workflows/verify-solutions.yml` in each system repo (claude-code, microservices)

```yaml
name: Verify Solutions
on: [push, pull_request]

jobs:
  verify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      
      - name: Install dependencies
        run: npm ci
      
      - name: Generate solution manifest
        run: npx 100xsystems audit-solutions claude-code track-typescript
      
      - name: Verify all solutions
        run: npx 100xsystems verify-solutions claude-code track-typescript
      
      - name: Check solution manifest vs actual
        run: |
          # Fail if manifest shows expected files but solution doesn't have them
          node -e "
            const m = require('./.solution-manifest.json');
            let errors = 0;
            for (const [slug, lesson] of Object.entries(m.lessons)) {
              for (const file of lesson.expectedFiles) {
                const solutionPath = 'track-typescript/' + /* find lesson dir */ + '/solution/' + file.path;
                if (!fs.existsSync(solutionPath)) {
                  console.error('❌ ' + slug + ': missing solution file ' + file.path);
                  errors++;
                }
              }
            }
            process.exit(errors > 0 ? 1 : 0);
          "
```

### 3.4 GitHub Webhook Integration

After E2E verification passes, sync the results to the website for a solution health dashboard:

```
POST /api/v1/solution-status
{
  "system": "claude-code",
  "track": "track-typescript",
  "results": [
    { "lesson": "lesson-intro-and-setup", "passed": true, "testsPassed": 6 },
    ...
  ],
  "sha": "abc123"
}
```

---

## 4. Implementation Plan

### Phase 1: Audit & Manifest (Week 1)

| Task | File | Est. |
|------|------|------|
| Create `SolutionManifest` type | `cli/src/actions/audit-solutions.ts` | 2h |
| Implement `parseTestExpectations()` — extract fileExists, fileMatches, dirExists calls from test file content | `cli/src/actions/audit-solutions.ts` | 4h |
| Implement `extractRequiredDeps()` — detect npm package imports in test files | `cli/src/actions/audit-solutions.ts` | 1h |
| Add `100xsystems audit-solutions` command | `cli/src/commands/audit-solutions.tsx` | 1h |

### Phase 2: Solution Verification (Week 2)

| Task | File | Est. |
|------|------|------|
| Implement `verifySolutions()` — cumulative copy + test runner | `cli/src/actions/verify-solutions.ts` | 6h |
| Integrate with test-runner executor (reuse its logic) | `cli/src/actions/verify-solutions.ts` | 2h |
| Handle cumulative file tracking (don't overwrite newer with older) | `cli/src/actions/verify-solutions.ts` | 2h |
| Add `100xsystems verify-solutions` command | `cli/src/commands/verify-solutions.tsx` | 1h |

### Phase 3: CI Integration (Week 3)

| Task | File | Est. |
|------|------|------|
| Create `verify-solutions.yml` workflow | `.github/workflows/verify-solutions.yml` (each system repo) | 2h |
| Add solution manifest assertion step | CI script | 1h |
| Create solution health dashboard endpoint | `website/app/api/v1/solution-status/route.ts` | 2h |

### Phase 4: Auto-generation (Week 4 — Stretch)

| Task | File | Est. |
|------|------|------|
| Implement `generate-solution` — create solution stubs from test expectations | `cli/src/actions/generate-solution.ts` | 4h |
| Add `100xsystems generate-solution <lesson>` command | `cli/src/commands/generate-solution.tsx` | 1h |
| Auto-fill missing solution files from test expectations | `cli/src/actions/generate-solution.ts` | 3h |

---

## 5. Key Design Decisions

### Decision 1: Cumulative vs Isolated Testing

**Chosen: Cumulative.** Each lesson builds on the previous. Testing lesson 5 in isolation would miss the fact that lesson 5 depends on files from lessons 1-4 being present and compiling. The cumulative approach:
- Starts fresh for lesson 1
- Copies solution 1 → typecheck → test
- Copies solution 1+2 → typecheck → test
- Copies solution 1+2+3 → typecheck → test
- ... (cumulatively builds up)

This catches cross-lesson import errors and regression bugs.

### Decision 2: Manifest vs Direct Test Scanning

**Chosen: Manifest.** Pre-generating a `.solution-manifest.json` from test files and comparing against solutions provides:
- **Explicit documentation** of what each test expects
- **Fast comparison** (no need to parse test files on every run)
- **Diff-friendly** — commit history shows when expectations change
- **Audit trail** — who changed what and when

### Decision 3: CLI Command vs Standalone Script

**Chosen: CLI Command.** Adding `verify-solutions` and `audit-solutions` as CLI commands (not standalone scripts) because:
- Reuses all existing infrastructure (SYSTEMS_DIR, test-runner, auth, registry sync)
- Single interface for users and CI
- Tests solutions from any system repo (not just claude-code)

### Decision 4: File Content Tracking

When copying solutions cumulatively, track file origin (which lesson provided it). If lesson 3's solution provides `src/cli.ts` and lesson 5 also provides `src/cli.ts`, the lesson 5 version wins (newer lesson = more complete solution). This is correct behavior because later lessons build on earlier ones.

```typescript
const fileOrigin = new Map<string, string>(); // filePath → lessonSlug

// Copy solution files, tracking origin
function copySolutionFiles(solutionDir: string, targetDir: string, fileOrigin: Map<string, string>, lessonSlug: string): void {
  for (const [relPath, content] of getFiles(solutionDir)) {
    fileOrigin.set(relPath, lessonSlug); // newer lesson overwrites origin
    writeFile(targetDir, relPath, content);
  }
}
```

---

## 6. Success Metrics

| Metric | Target | How to Measure |
|--------|--------|----------------|
| Solution ↔ test drift caught | 100% before merge | CI fails on drift |
| Time to verify all solutions | < 5 min | GitHub Action runtime |
| False positives | 0 | Manual verify of first 10 failures |
| Solution health dashboard accuracy | 100% | One-to-one with CI results |

---

## 7. Future Enhancements

- **Solution status page on website** — green/red/yellow per lesson showing solution health
- **Auto-fix PRs** — when test expectations change, auto-generate solution updates and open PRs
- **Per-track dashboard** — "claude-code track-typescript: 8/8 solutions healthy" badge
- **AI-assisted solution generation** — use LLM to generate solution stubs from lesson.md + test expectations

---

*Generated: July 19, 2026*
