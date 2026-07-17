---
title: "CAP Theorem"
order: 1
difficulty: "Intermediate"
tags: ["distributed-systems", "consistency", "availability"]
---

The CAP Theorem (also known as Brewer's Theorem) states that a distributed data store can only provide two of the following three guarantees simultaneously:

- **Consistency** — Every read receives the most recent write or an error
- **Availability** — Every request receives a (non-error) response
- **Partition Tolerance** — The system continues to operate despite failures

## Why It Matters

In distributed systems, network partitions are inevitable. You must choose between CP and AP.

## Trade-offs

| Choice | Behavior | Examples |
|--------|----------|---------|
| **CP** | System returns errors during partition | HBase, MongoDB (default) |
| **AP** | System returns stale data during partition | Cassandra, DynamoDB |

## Related

- [BASE](/principles/base)
- [ACID](/principles/acid)
