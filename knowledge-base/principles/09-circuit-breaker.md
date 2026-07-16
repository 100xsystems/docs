---
title: "Circuit Breaker"
order: 9
difficulty: "Intermediate"
tags: ["resilience", "microservices", "fault-tolerance"]
---

Prevents cascading failures in distributed systems by failing fast.

## States

| State | Behavior |
|-------|----------|
| **Closed** | Normal operation |
| **Open** | Failures detected — requests fail fast |
| **Half-Open** | Testing recovery — limited requests pass |

## Implementations

Resilience4j (Java), Hystrix (Netflix), Polly (.NET)
