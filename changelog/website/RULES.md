# 100xSystems — AI Agent Rules

> **Purpose**: This file encodes the core principles, constraints, and behavioral rules that every AI agent (including future selves) must follow when working on 100xSystems. Read this first before making any changes.

---

## 1. The North Star

**100xSystems is NOT a feature factory. It is a PRODUCT.**

> _"We are not going to focus on features. We are going to focus on the product. The actual business model, the business implementation, and how this product can be helpful to millions of engineers and computer science people from day one."_ — Founder

Every decision must be evaluated against:
- **Does this help a beginner engineer learn effectively?**
- **Does this keep the content front and center?**
- **Does this simplify rather than complicate?**

## 2. Content Over Everything

**Content is king. The platform serves the content, not the other way around.**

- Markdown files are the source of truth for ALL educational content
- Images should be co-located with markdown or stored in a predictable parallel structure
- No database-dependent content rendering — content must be readable from files alone
- Fancy UI features that distract from content are REJECTED
- The content must have DEPTH — but organized into digestible chapters

## 3. Simplicity Mandate

**Simple > Complex. Always.**

- No over-engineered admin dashboards
- No complex CMS systems (no React-Quill, no GitHub-as-CMS storage)
- No AI-powered knowledge graphs that replace good navigation
- No unnecessary real-time features
- Prefer static generation (ISR/SSG) over SSR where possible
- Prefer file-based content over database-driven content

## 4. Odin Project Principles (Adapted)

**We follow The Odin Project model, NOT the CodeCrafters model (for content).**

| Principle | Implementation |
|-----------|---------------|
| Open source curriculum | All markdown content lives in the repo |
| Project-based learning | Every module ends with a project |
| Knowledge checks | Each chapter has review questions at the end |
| Community submissions | Students submit project links, viewable by others |
| Donation-funded | No paywalls — Open Collective model |
| Beginner-first | Content starts from absolute zero |
| Contributor-driven | Well-documented contribution guidelines |

## 5. CodeCrafters Inspirations (Limited)

**Some CodeCrafters ideas are good, but applied differently:**

- "Build your own X" projects ARE valuable — but integrated INTO the curriculum, not as standalone challenges
- Testing infrastructure is valuable — but NOT required for MVP
- The concept of "stages" for project breakdown is useful

## 6. The Depth-Relevance-Updates Triangle

**Every content decision must balance these three:**

```
        DEPTH
        /  \
       /    \
      /      \
RELEVANCE —— UPDATES
```

- **Depth**: Content must be deep, covering fundamentals
- **Relevance**: Content must be digestible, not overwhelming
- **Updates**: Content must be easy to update as technologies change

**Rule**: RELEVANCE is the key. If content isn't relevant to the student, depth and updates don't matter.

## 7. Architectural Constraints

- **Next.js**: App Router (migrating from Pages Router), latest version
- **Styling**: Tailwind CSS + CSS Modules (existing patterns respected)
- **Content**: Markdown files in `/content/` directory
- **Components**: Atomic design system in `__components/`
- **Storybook**: All new components documented in `_storybook/`
- **State**: Minimal — prefer URL state over global state
- **Data**: Supabase only for auth + community features, NOT for content

## 8. What NOT To Build

DO NOT build:
- Live code editors (React-Quill, CodeMirror, etc.)
- AI-powered chat bots as a core feature
- Knowledge graphs / mind maps as navigation
- Complex CMS admin dashboards
- Real-time collaboration features
- Video streaming platforms
- Interactive coding environments (for MVP)

## 9. Contribution Philosophy

**The curriculum and website are BOTH open source.**

- Content is written in Markdown with YAML frontmatter
- Anyone can submit PRs to fix bugs, update content, improve lessons
- The repo is the single source of truth — no external database for content
- Contribution guidelines are clear and welcoming to beginners

## 10. Immediate Priorities

1. Create `/content/` directory with markdown-based curriculum structure
2. Build an App Router layout that renders markdown content
3. Build project submission system (links + gallery)
4. Build progress tracking (checkboxes per lesson)
5. Add knowledge checks at the end of each chapter
6. Open Collective donation integration
