# 100xSystems Overview

## What It Is

100xSystems is a learning platform where engineers build real software systems from scratch. Not tutorials, not courses — you write the code, and our CLI validates it against behavioral test suites.

## Core Components

### 1. CLI (`100xsystems/cli`)
The command-line tool is the primary interface. Available as `@100xsystems/cli` on npm.

```bash
npm install -g @100xsystems/cli

# Discover systems
100xsystems list

# Start building
100xsystems init claude-code
cd claude-code-implementation

# Validate your work
100xsystems validate

# Submit for review
100xsystems submit
```

### 2. Website (`100xsystems/website`)
Built with Next.js 16, deployed as a static site. Displays systems, tracks, modules, and lessons. Syncs curriculum from the registry during build.

### 3. Registry (`100xsystems/registry`)
A lightweight JSON index of all available learning systems. Both the CLI and website read from this to discover systems.

### 4. System Repositories
Each system has its own repository containing curriculum, tests, and assets:
- `100xsystems/claude-code` — AI coding agent
- `100xsystems/microservices` — Distributed systems

## Architecture

```
User runs: 100xsystems init <system>
                │
                ▼
         CLI reads registry.json
                │
                ▼
         Clones system repo into ~/.cache/100xsystems/repos/
                │
                ▼
         Scaffolds project from curriculum templates
                │
                ▼
         User builds the system
                │
                ▼
         CLI validates with behavioral tests
                │
                ▼
         User submits via PR
                │
                ▼
         Mentor reviews → merges → certificate issued
```

## Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Repository structure | Per-system repos | Easy for contributors, natural ownership boundaries |
| System discovery | Registry JSON | Single source of truth, no GitHub API needed |
| Validation | Behavioral test suites | Real verification, not just file existence checks |
| Test framework | Vitest (pluggable) | JUnit, Go test, Cargo test planned for other languages |
| Lesson format | Folder-based | Self-contained: lesson.md + tests/ + assets/ |
| Website build | Clone from registry | Auto-discovers new systems, zero-config |
| CLI caching | ~/.cache/100xsystems/repos/ | Offline-friendly, fast subsequent runs |

## Contributing

See the [Contributing Guide](https://github.com/100xsystems/.github/blob/main/CONTRIBUTING.md).
