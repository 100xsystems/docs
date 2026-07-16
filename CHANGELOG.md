# Changelog

All notable changes to 100xSystems will be documented in this file.

## [1.0.0] — 2026-07-16

### Added

#### Organization
- Created 100xSystems GitHub organization with 8 repositories
- Established `.github` repo with community health files (CODE_OF_CONDUCT, SECURITY, CONTRIBUTING)
- Created maintainers team with org-level repository access
- Enabled branch protection on all repos (1 reviewer, linear history, enforce admins)
- Added MIT license to all repositories
- Added repo descriptions for discoverability
- Added org profile README at github.com/100xsystems
- Created system-template repo for easily bootstrapping new systems
- Added standardized issue labels (curriculum, needs review, blocked)

#### CLI
- `100xsystems init` — Scaffold projects from any system in the registry
- `100xsystems validate` — 3-level validation (structure, lesson validators, spec)
- `100xsystems submit` — Package and submit code for review
- `100xsystems list` — Browse available systems
- `100xsystems progress` — Track per-module completion
- Registry-based system discovery (auto-syncs on init and list)
- Test-runner executor: runs real Vitest test suites in isolated temp directories
- Pluggable executor system (file-exists, file-contains, cli-command, regex, http, docker, npm-test, test-runner)
- Folder-based lesson format support (lesson.md + tests/behavior.test.ts)

#### Curriculum
- **Claude Code** (TypeScript track): 7 lessons across 3 modules + quizzes + challenge
- **Claude Code** (Java track): 2 lessons in module 1
- **Microservices** (Spring Boot track): 7 lessons across 4 modules
- Knowledge base: principles, patterns, tools, technologies

#### Website
- Next.js 16 with SSG (Static Site Generation)
- Registry-based curriculum sync at build time
- System browse, track hierarchy, lesson navigation
- Community page with submission data
- Certificate verification page

#### Migration
- Extracted all systems into separate repositories
- Created 100xsystems/registry as the central system index
- Updated CLI reader to support cache-based system discovery
- Updated website build to clone systems from registry

### Changed
- All lessons converted from flat `.md` files to folder-based format
- Tests moved from root to `tests/behavior.test.ts` subfolder
- CLI SYSTEMS_DIR now resolves cache → local → env var
- Init command auto-syncs from registry on startup
- List command auto-syncs from registry before displaying

### Security
- Branch protection enforced on all repositories

## [0.1.0] — 2026-07-10

### Added
- Initial CLI implementation with Ink-based UI
- Basic project scaffolding for TypeScript tracks
- File existence and content validation executors
- Initial curriculum structure
- PKCE-based GitHub authentication
