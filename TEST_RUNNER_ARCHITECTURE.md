# Test Runner Architecture — Behavioral Validation System

> **Date:** July 16, 2026
> **Status:** Implemented (Phase 1 — Vitest for TypeScript)
> **Architect:** Buffy (Freebuff AI Agent)

---

## The Problem

The old validation system only checked surface-level conditions:
- `file-exists`: "Does the file exist?"
- `file-contains`: "Does the string exist in the file?"
- `cli-command`: "Does the command exit with code 0?"

**None of these actually test BEHAVIOR.** You could submit a PR with broken code and all checks would pass because no one was actually running the code and verifying the output.

---

## The Solution: Per-Lesson Test Suites

Each lesson now has a **real test file** that:
1. Imports the user's compiled modules from `dist/`
2. Calls exported functions with controlled test inputs
3. Asserts the output matches expectations
4. Includes cumulative checks from previous lessons (no regressions)

### Lesson Folder Structure

Every lesson is now a **folder** containing `lesson.md` + `test.spec.ts`:

```
module-1-cli-foundations/
├── 01-lesson-intro-and-setup/       ← Folder-based lesson
│   ├── lesson.md                    ← Lesson content (MD with frontmatter)
│   └── test.spec.ts                 ← NEW: Behavioral test suite (Vitest)
├── 02-lesson-build-cli-tool/
│   ├── lesson.md
│   └── test.spec.ts
├── 03-lesson-agent-loop/
│   ├── lesson.md
│   └── test.spec.ts
├── 04-lesson-implement-tools/
│   ├── lesson.md
│   └── test.spec.ts
├── quiz.md                          ← Legacy (unchanged)
└── challenge.md                     ← Legacy (unchanged)
```

### Lesson Frontmatter

The `validation:` block in each lesson.md now references `test-runner` instead of individual executors:

```yaml
validation:
  - type: test-runner
    test_file: "test.spec.ts"       # In the same folder as lesson.md
    framework: vitest               # vitest | junit | go-test | cargo-test
    timeout: 120000                 # 2 minutes for install + run
    expected_passes: 6              # Optional: minimum passing tests
```

---

## New Executor: `test-runner`

**File:** `cli/src/executors/test-runner.executor.ts`

### Runtime Flow

```
1. Read test.spec.ts from curriculum/.../{lesson-slug}/test.spec.ts
2. Create temp dir /tmp/100x-test-vitest-{random}/
3. Copy user's entire project (excl. node_modules, .git, dist) into temp dir
4. Copy test.spec.ts into temp dir
5. Inject vitest dependency into temp package.json
6. npm install (in temp dir — fast from npm cache)
7. npx vitest run --reporter json
8. Parse JSON → map test results to ValidationResult[]
9. Clean up temp dir (finally block — always runs)
```

### Key Design Decisions

| Decision | Choice | Why |
|----------|--------|-----|
| **Isolation** | Temp directory | Zero pollution of user's project |
| **Dependencies** | Auto-install vitest | No assumptions about user's env |
| **Framework** | Pluggable dispatch | Same executor handles all languages |
| **Test ownership** | Curriculum-owned | User can't cheat by modifying tests |
| **Cumulative checks** | Each test includes previous | Prevents regressions |
| **Output parsing** | Vitest JSON reporter | Structured, machine-parseable |

### Supported Frameworks (Planned)

| Framework | Status | Language | CLI Command |
|-----------|--------|----------|-------------|
| `vitest` | ✅ **Done** | TypeScript/JavaScript | `npx vitest run --reporter json` |
| `junit` | 🔜 Planned | Java | `mvn test` (surefire) |
| `go-test` | 🔜 Planned | Go | `go test -json ./...` |
| `cargo-test` | 🔜 Planned | Rust | `cargo test --format json` |

---

## 3-Level Validation System

| Level | Name | What It Checks | Source |
|-------|------|----------------|--------|
| **L1** | Project Structure | `100xsystems.json`, README, package.json, src/ directory, git repo | `checkLevel1()` in `validate.ts` |
| **L2** | Lesson Validators | **NEW: Real test suites** via `test-runner` executor | `lesson.md` frontmatter → `test.spec.ts` |
| **L3** | Spec-Defined | File existence, doc sections, test passes | `SPECIFICATION.md` |

---

## Files Changed

### New Files

| File | Purpose |
|------|---------|
| `cli/src/executors/test-runner.executor.ts` | The test-runner executor — vitest framework support |
| `curriculum/.../01-lesson-intro-and-setup/test.spec.ts` | 6 behavioral tests for lesson 1 |
| `curriculum/.../02-lesson-build-cli-tool/test.spec.ts` | 6 behavioral tests for lesson 2 |
| `curriculum/.../03-lesson-agent-loop/test.spec.ts` | 6 behavioral tests for lesson 3 |
| `curriculum/.../04-lesson-implement-tools/test.spec.ts` | 6 behavioral tests for lesson 4 |
| `curriculum/.../01-lesson-llm-integration/test.spec.ts` | 5 behavioral tests for lesson 5 |
| `curriculum/.../02-lesson-context-management/test.spec.ts` | 3 behavioral tests for lesson 6 |
| `curriculum/.../01-lesson-tool-system/test.spec.ts` | 3 behavioral tests for lesson 7 |

### Modified Files

| File | Change |
|------|--------|
| `cli/src/executors/registry.ts` | Registered `test-runner` executor |
| `cli/src/executors/index.ts` | Exported `TestRunnerExecutor` |
| `cli/src/reader/lesson-reader.ts` | `getModuleLessons()` supports folder/lesson.md format |
| `cli/src/actions/validate.ts` | `findLessonsWithValidators()` and `findAllLessonFiles()` support folder format |

---

## How to Test

### Prerequisites

1. The CLI must be built and linked:
```bash
cd cli
npm run build
npm link
```

2. You need a scaffolded project to test against. If you don't have one:
```bash
cd /tmp
mkdir test-project && cd test-project
100xsystems init claude-code --output claude-code-agent
cd claude-code-agent
# Complete the lesson requirements (create src/, package.json, etc.)
```

### Test 1: Verify the new executor is registered

```bash
# Check that test-runner appears in the available executor types
cd cli
node -e "
const { registry } = require('./dist/executors/registry.js');
console.log('Registered executors:', registry.types());
console.log('test-runner registered:', registry.has('test-runner'));
"
```

Expected output:
```
Registered executors: file-exists, http, file-contains, regex, npm-test, cli-command, docker, test-runner
test-runner registered: true
```

### Test 2: Validate a lesson with the test-runner

Navigate to your scaffolded project and run validate:

```bash
cd /path/to/your/project
100xsystems validate
```

1. Select the first lesson (Introduction & Project Setup)
2. The CLI will show the validation is running
3. The test-runner will:
   - Copy your project to a temp directory
   - Copy the test.spec.ts from the curriculum
   - Install vitest
   - Run the tests
   - Report pass/fail per test

### Test 3: Test with a real project

If you have a project that already has code written, you'll see real behavioral tests pass:

```bash
cd /path/to/your/claude-code-implementation
100xsystems validate
```

Select a lesson and watch the tests run. Each test that passes means the code BEHAVES correctly — not just "the file exists."

### Test 4: Verify isolation

Run validate twice in succession. Each run creates a fresh temp directory, so no state leaks between runs.

```bash
100xsystems validate   # First run — installs vitest
100xsystems validate   # Second run — faster (vitest cached)
```

### Test 5: Test with broken code (to see failure messages)

Introduce a deliberate error:

```bash
cd /path/to/your/project
# Break something
echo 'export const broken = true;' > src/index.ts
100xsystems validate
```

You should see test failures with clear error messages showing what went wrong.

---

## Architecture Diagram

```
User runs: 100xsystems validate
                │
                ▼
      validate.tsx (Ink UI)
                │
                ▼
      runValidation(projectDir, config, lessonSlug)
                │
                ├── Level 1: checkLevel1()
                │     └── Config, README, package.json, src/, git
                │
                ├── Level 2: runLessonValidatorsFromCurriculum()
                │     └── Read lesson.md frontmatter
                │           └── validation:
                │                 └── type: test-runner
                │                       │
                │                       ▼
                │               TestRunnerExecutor.execute()
                │                     │
                │                     ├── 1. Create /tmp/100x-test-{random}/
                │                     ├── 2. Copy user's src/, package.json, tsconfig.json
                │                     ├── 3. Copy curriculum's test.spec.ts
                │                     ├── 4. Inject vitest dep
                │                     ├── 5. npm install vitest
                │                     ├── 6. npx vitest run --reporter json
                │                     ├── 7. Parse JSON → ValidationResult[]
                │                     └── 8. Clean up temp dir
                │
                └── Level 3: runSpecChecksFromCurriculum()
                      └── SPECIFICATION.md checks
```
