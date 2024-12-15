+++
title = "Should We Move to a Microservices Architecture?"
series = ["Design Practices"]
date = "2024-12-14"
+++

## Literature Review of Key Benefits
This section highlights the key benefits of microservices architecture according to several sources.

- According to Martin Fowler’s article [Microservice Trade-Offs](https://martinfowler.com/articles/microservice-trade-offs.html):
  - Strong Module Boundaries
  - Independent Deployment
  - Technology Diversity
- According to Chapter 1 of [Building Microservices (First Edition)](https://www.oreilly.com/library/view/building-microservices/9781491950340/):
  - Resilience
  - Scalability
  - Organizational Alignment

### Strong Module Boundaries
> Chunks of software that are decoupled from each other [...] Good modular structure is useful in any program, but becomes exponentially more important as the software grows in size.

Modularity is a key principle of microservices architecture. Each service should be small and focused on doing one thing well. This approach aligns with the [Low Coupling, High Cohesion principle](https://enterprisecraftsmanship.com/posts/cohesion-coupling-difference/), ensuring that services function as independent units with minimal interdependencies. Each service is responsible for a clearly defined piece of functionality.

To achieve this, it’s essential to define service boundaries that align with business domains, making it clear where each piece of functionality belongs.

### Independent Deployment
> A key principle of microservices is that [services are components](https://martinfowler.com/articles/microservices.html#ComponentizationViaServices) and thus are independently deployable.

However, many attempts at microservices fail at independent deployment. Multiple services sometimes require careful coordination to be released together, which undermines one of the fundamental benefits of microservices.

### Technology Diversity
> Since each microservice is an independently deployable unit, you have considerable freedom in your technology choices within it. Microservices can be written in different languages, use different libraries, and use different data stores.

This flexibility allows teams to evolve their parts of the system independently, adopting technologies that best fit their needs. It gives you the freedom to choose the most suitable stacks and to release new versions at will.

### Resilience
> If one component of a system fails, the failure doesn't cascade.

By running multiple instances of the same service, the system can continue to operate in a degraded mode, ensuring overall functionality even when individual services encounter issues.

However, microservices also introduce new points of failure. Networks and machines can fail, so you must plan for this and understand the impact on end users.

### Scalability
> With a large, monolithic service, we have to scale everything together. With smaller services, we can scale only those services that need it.

When considering scalability, ask yourself: What problem are we solving? Do we have enough data to justify splitting services for performance or scaling reasons? Can our system scale and remain resilient simultaneously? Microservices encourage distributed systems designed for size and resilience, but this introduces complexities: backpressure, circuit breakers, queues, jitter, and sensible timeouts all become essential considerations.[^4]

### Organizational Alignment
> We also know that smaller teams working on smaller codebases tend to be more productive.

[Conway’s Law](https://martinfowler.com/bliki/ConwaysLaw.html) suggests that software structures often reflect the communication structures of the organizations that build them. Aligning service ownership with teams that share physical proximity and focus on the same bounded contexts is crucial. Autonomous microservices teams should be loosely coupled, responsible for a specific business function, and free to develop, deploy, and scale services as they see fit.

## Literature Review of Key Challenges
It’s important to maintain a balanced view. In the following sections, we’ll address the potential drawbacks and challenges of implementing microservices.

- According to Martin Fowler’s [Microservice Trade-Offs](https://martinfowler.com/articles/microservice-trade-offs.html):
  - Distribution
  - Eventual Consistency
  - Operational Complexity

### Distribution
> Distributed systems are harder to program, since remote calls are slow and always at risk of failure.

- Performance issues: Remote calls add latency.
- Asynchronous programming is challenging: It’s more difficult to get right and much harder to debug, yet sometimes necessary for acceptable performance.
- [Fallacies of Distributed Computing](https://en.wikipedia.org/wiki/Fallacies_of_distributed_computing)

Related blog article: [My microservice architecture is timing out](https://dabase.com/blog/2021/Microservice-timeouts/)

### Eventual Consistency
> With a monolith, you can update multiple things together in a single transaction. Microservices require multiple resources to update, and distributed transactions.

Microservices often lead to data fragmentation across multiple databases or storage systems, each optimized for a particular service. This makes maintaining data consistency and integrity across the entire system more complex.[^3]

The [SAGA](https://microservices.io/patterns/data/saga.html) pattern is commonly used to handle eventual consistency.

### Operational Complexity
> You need a mature operations team to manage numerous services, which are redeployed regularly.

Effective operations require a DevOps culture, with greater collaboration between developers, operations, and everyone else involved in software delivery. Microservices demand more operational competence than monoliths.

## The Golden Rule
*Strong Module Boundaries* and *Independent Deployment* principles emphasize a critical point[^1]:
> Without decoupling, everything breaks down for us. **The golden rule: Can you make a change to a service and deploy it by itself without changing anything else?**

To achieve this, you must:
- Model your services correctly (e.g., using [Bounded Contexts](https://martinfowler.com/bliki/BoundedContext.html)).
- Get the APIs right.

### Start by Drawing Boundaries
> If you are unable to design a monolith that is cleanly divided into components, you will also be unable to design a microservice system.[^2]

## Give Yourself Time to Reflect
Microservices architecture has distinct advantages and challenges. While it has significantly influenced software design over the past decade, adopting it demands careful consideration. The following checklist, reflecting my own perspective, can help guide your decision-making process.

| **Theme**                | **Question**                                                                                                                                | **Yes/No** |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------- | ---------- |
| Strong Module Boundaries | Do we have domain experts?                                                                                                                  |            |
| Independent Deployment   | Can we easily set up the necessary CI/CD pipelines, containerization, and orchestration for independent deployments?                        |            |
|                          | Do we have the operational capacity, as well as the monitoring, logging, and tracing capabilities, to manage multiple independent services? |            |
| Technology Diversity     | Do we need flexibility to use different technology stacks, frameworks, or databases for different parts of the system?                      |            |
| Structural Organization  | Are our teams structured so that each can own a service end-to-end without heavy cross-team dependencies?                                   |            |
|                          | Does the team have the ability to develop, deploy, and scale services as they see fit?                                                      |            |
| Resilience               | Is it crucial to maintain partial system functionality if one component fails?                                                              |            |
| Scalability              | Do different parts of the system have distinct performance or scaling requirements that justify targeted scaling?                           |            |
| Human Resources          | Do we have the talent to handle the increased complexity in infrastructure, deployment, and maintenance?                                    |            |
|                          | Do we have the budget to handle the additional complexity and operational costs (e.g., tooling, cloud infrastructure)?                      |            |

<br />

## The Modular Monolith as an Alternative
- In reality, few companies require extensive scalability or resilience. Often, the primary goal is to improve software decoupling.
- In many organizations, software becomes a [Big Ball of Mud](https://wiki.c2.com/?BigBallOfMud), difficult to refactor and maintain, and hard to sustain productivity over time.
- Many decision-makers believe that adopting microservices will solve all their problems. However, the core issue often lies in achieving proper **modularity**.

![](https://raw.githubusercontent.com/ttulka/blog-assets/master/monolith/types-of-systems.png)

Sam Newman, in [Building Microservices (2nd Edition)](https://www.oreilly.com/library/view/building-microservices-2nd/9781492034018/), endorses using microservices only after becoming convinced of their benefits, rather than defaulting to them in every project.

> A monolithic architecture is a choice, and a valid one at that. I’d go further and say that in my opinion it is the sensible default choice as an architectural style. In other words, I am looking for a reason to be convinced to use microservices, rather than looking for a reason not to use them.

### Taking Advantages of Both Worlds
A modular monolithic approach addresses modularity concerns effectively.
> There's no reason why a monolithic system shouldn't have a good modular structure.

By emphasizing clear [separation of concerns](https://effectivesoftwaredesign.com/2012/02/05/separation-of-concerns/), different functionalities can be encapsulated within distinct, well-defined modules. These modules interact through in-memory calls, maintaining efficient communication without the overhead of network calls. A consistent codebase and shared runtime environment help enforce boundaries while preserving simplicity and coherence across the system.

## Further Reading
**General Articles**
- [Microservices Guide](https://martinfowler.com/microservices/)
- [Microservice Trade-Offs - Martin Fowler](https://martinfowler.com/articles/microservice-trade-offs.html)
- [Monolith First - Martin Fowler](https://martinfowler.com/bliki/MonolithFirst.html)
- [The microservice architecture is meant to simplify and accelerate development, but only when done correctly](https://microservices.io/post/architecture/2023/07/06/msa-is-meant-to-simplify-development.md.html)

**What’s Wrong with Microservices**
- [Death by a Thousand Microservices](https://renegadeotter.com/2023/09/10/death-by-a-thousand-microservices.html)
- [You Probably Don’t Need Microservices](https://www.thrownewexception.com/you-probably-dont-need-microservices/)

**Modular Monolith Approach**
- [Modular Monolith Series](https://www.kamilgrzybek.com/blog/series/modular-monolith)
- [Good and Bad Monolith](https://blog.ttulka.com/good-and-bad-monolith/)
- [Making Modular Monoliths Work](https://sookocheff.com/post/architecture/making-modular-monoliths-work/)

[^1]: Building Microservices (First Edition) page 3  
[^2]: https://softwareengineering.stackexchange.com/a/339234/385571  
[^3]: https://www.shopify.com/enterprise/blog/disadvantages-microservices#  
[^4]: [Death by a Thousand Microservices](https://renegadeotter.com/2023/09/10/death-by-a-thousand-microservices.html)