---
title: "DRY (Don't Repeat Yourself)"
order: 2
difficulty: "Beginner"
tags: ["code-quality", "clean-code", "best-practices"]
---

**DRY** states every piece of knowledge must have a single, unambiguous, authoritative representation within a system.

## Why It Matters

- **Maintenance nightmares** — fix the same bug in 10 places
- **Inconsistency** — one copy gets updated, others don't
- **Cognitive overhead** — harder to understand the system

## How to Apply

1. Extract repeated logic into shared functions
2. Use data structures as single source of truth
3. Leverage abstractions for repeated patterns

## Related

- [KISS](/principles/kiss)
- [YAGNI](/principles/yagni)
- [SOLID](/principles/solid)
