---
title: "Factory Pattern"
order: 1
difficulty: "Beginner"
tags: ["creational", "design-patterns", "oop"]
---

Provides an interface for creating objects without specifying their concrete classes.

## When to Use

- Class can't anticipate the class of objects it must create
- Class wants subclasses to specify the objects it creates
- You want to encapsulate object creation logic

## Example

```java
// Instead of: new Car(), new Truck() everywhere
VehicleFactory.create("car");
VehicleFactory.create("truck");
```
