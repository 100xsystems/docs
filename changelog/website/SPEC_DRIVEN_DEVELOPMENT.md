# Spec-Driven Development — 100xSystems

## What Is This?

Every significant change to 100xSystems must follow **spec-driven development (SDD)**. This means:

1. **Write the spec first** — before writing code, document WHAT you're building and WHY
2. **Get alignment** — the spec must reference project principles (docs/RULES.md)
3. **Implement** — code follows the spec exactly
4. **Validate** — test that implementation matches the spec

---

## Spec Template

Every feature or significant change MUST include a spec document following this template:

```markdown
# Spec: [Feature/Change Name]

## Problem Statement
[What problem does this solve? Reference user pain points.]

## Alignment with Principles
- [ ] Content-first: [How does this serve content delivery?]
- [ ] Simplicity: [How is this simpler than alternatives?]
- [ ] Beginner-friendly: [How does this help beginners?]

## Specification

### User Stories
- As a [user type], I want to [action] so that [benefit].

### Requirements (Functional)
1. [Requirement 1]
2. [Requirement 2]

### Requirements (Non-Functional)
1. Performance: [e.g., "Page loads < 2s"]
2. Accessibility: [e.g., "WCAG 2.1 AA"]
3. Responsiveness: [e.g., "Works on mobile"]

### Data Model
[If applicable, describe data structures]

### UI/UX Design
[Wireframe or description of user interface]

### API/Interfaces
[If applicable, describe API contracts]

## Implementation Plan

### Files to Create/Modify
1. `path/to/file.ts` — Reason for change

### Dependencies
- [Package or system dependencies]

### Migration Steps
[If breaking changes, describe migration]

## Open Questions
[Questions that need resolution before implementation]

## Revisions
- v1: Initial spec [date]
```

---

## Code Review Checklist

Every PR/code change must pass these checks:

### Content Changes
- [ ] Content is in markdown format
- [ ] YAML frontmatter is complete (title, description, order, knowledge_checks)
- [ ] Images are in correct parallel directory
- [ ] Knowledge checks are present at end of chapter
- [ ] Chapter is digestible (< 30 min read)

### Component Changes
- [ ] Component follows atomic design hierarchy
- [ ] Component has Storybook story
- [ ] Component accepts className prop for extension
- [ ] Component is accessible (keyboard, screen reader)
- [ ] Props are typed with TypeScript
- [ ] No unnecessary re-renders (use memo where beneficial)
- [ ] Component is documented with JSDoc

### Architecture Changes
- [ ] Follows App Router conventions (no Pages Router)
- [ ] Server components by default, client components when needed
- [ ] No new dependencies without justification
- [ ] Content is file-based, not database-driven
- [ ] Follows existing project conventions

---

## Decision Records

Major architectural decisions MUST be documented in `docs/adr/` with this format:

```markdown
# ADR-XXX: [Title]

## Status
[Proposed | Accepted | Deprecated | Superseded]

## Context
[What is the context and why was this decision needed?]

## Decision
[What was decided?]

## Consequences
[What are the trade-offs, pros and cons?]

## Alternatives Considered
[What else was considered and why was it rejected?]
```
