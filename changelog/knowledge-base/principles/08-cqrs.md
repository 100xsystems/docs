---
title: "CQRS"
order: 8
difficulty: "Advanced"
tags: ["architecture", "patterns", "event-sourcing"]
---

Separates read and write operations into different models.

## Why CQRS

- Optimized models for reads and writes independently
- Independent scaling of read/write workloads
- Different authorization for commands and queries

## When to Use

- Complex domains with different read/write patterns
- High-traffic systems where reads > writes
