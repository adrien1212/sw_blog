+++
title = "Low coupling and High cohesion"
series = ["Design Practices"]
date = "2023-08-25"
+++

In order to prolong the value of the software for a long period of time you have to make it respond easier to change. The good thing is that there are a few guidelines that you can follow that can help you out.

One way of achieving this is to structure software in line with the following guideline 

> Create components with low coupling and high cohesion

At its core, **high cohesion** involves **grouping related elements** of a codebase together in one location. Conversely, **low coupling** focuses on maximizing the **separation between unrelated components** of the codebase.

## Components
A component can be a function, a class, a namespace (a package), a micro-service, a whole monolith, or whatever construct as soon as it has an inside and an outside separated by a boundary. This boundary allows us to isolate some part of a component from other component.

By defining a **component at different layers of abstraction**, we demonstrate that coupling and cohesion are valuable concepts at every level.

## Coupling and Cohesion
- **Coupling** is the degree of interdependence between software modules.

- **Cohesion** in software engineering is the degree to which the elements of a certain module belong together

When an image is worth a thousand words has its meaning, please consider the following pictures

![Tight coupling, Low cohesion](../images/low_coupling_high_cohesion/lchc.png)
This first drawing illustrates Tight Coupling, Low Cohesion, a structural design characteristic commonly associated with Big Ball of Mud architecture.

![Tight coupling, High cohesion](../images/low_coupling_high_cohesion/lchc2.png)
The second picture represents a Layered Architecture (Tight coupling, High cohesion), where the developer organizes components into controllers, services, and repository layers, but without incorporating business logic.
```
- fr.adriencaubel.service
    userService
    notificationService
- fr.adriencaubel.repository
    userRepository
    notificationRepository
- fr.adriencaubel.controller
    userController
    notificationController
```

![Loose coupling, High cohesion](../images/low_coupling_high_cohesion/lchc3.png)

This third picture shows the ideal scenario, where components are organized with low coupling and high cohesion. This takes place when the boundaries between different component selected correctly (following Domain Driven Design). The following structure incorporates Layered Architecture with DDD.

```
- fr.adriencaubel.user
    userService
    userRepository
    userController
- fr.adriencaubel.notification
    notificationRepository
    notificationService
    notificationController
```
### Cohesion first
High Cohesion often leads to Low Coupling, as you improve cohesion, coupling often decreases naturally and help you to avoid Extreme Decoupling by striving for extremely low coupling without considering cohesion.

### Give me the advantages

By introducing this guideline in your architecture and design you improve :
- Maintainability of the code base : you'll have a clear model, a clear representation of your code base. It will be easier to modify and maintain.
- Team autonomy : different teams can work on different parts of the codebase, without stepping on each other's toes. If the high-level modules are highly cohesive and loosely coupled, changes to one module will not affect the others.
- Reutilization : use one part of your codebase on another.

## Further reader
- https://blog.ttulka.com/how-cohesion-and-coupling-correlate/
- https://enterprisecraftsmanship.com/posts/cohesion-coupling-difference/
- https://thevaluable.dev/cohesion-coupling-guide-examples/