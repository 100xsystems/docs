---
title: "Decorator Pattern"
order: 6
difficulty: "Intermediate"
tags: ["structural", "design-patterns", "extension"]
---

Attaches additional responsibilities to an object dynamically. A flexible alternative to subclassing.

## Example

```java
Coffee coffee = new SimpleCoffee();
coffee = new MilkDecorator(coffee);   // adds milk
coffee = new SugarDecorator(coffee);  // adds sugar
```

## When to Use

- Adding responsibilities to individual objects, not entire classes
- When subclassing creates too many combinations
