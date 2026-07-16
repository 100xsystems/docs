---
title: "Singleton Pattern"
order: 2
difficulty: "Beginner"
tags: ["creational", "design-patterns", "oop"]
---

Ensures a class has only one instance and provides a global point of access to it.

## When to Use

- Exactly one instance is needed
- Controlled access to shared resource (DB connection pool)
- Logging, caching, thread pools

## Caveats

- Can make code harder to test
- Consider dependency injection instead
- Thread safety requires careful implementation
