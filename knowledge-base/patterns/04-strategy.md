---
title: "Strategy Pattern"
order: 4
difficulty: "Intermediate"
tags: ["behavioral", "design-patterns", "algorithms"]
---

Defines a family of algorithms, encapsulates each one, and makes them interchangeable.

## When to Use

- Multiple classes that differ only in behavior
- Different variants of an algorithm
- Conditional statements that select behavior

## Example

A `PaymentProcessor` using `CreditCardStrategy`, `PayPalStrategy`, or `CryptoStrategy` interchangeably.
