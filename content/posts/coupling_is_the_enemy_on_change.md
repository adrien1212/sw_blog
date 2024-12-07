+++
title = "Coupling is the enemy on change"
series = ["Design Practices"]
date = "2024-11-25"
+++

Decoupling your code makes it significantly easier to modify, scale, and maintain over time. In our [Low Coupling and High Cohesion](../low_coupling_high_cohesion) article, we explore how cohesion plays a vital role in organizing projects effectively. Here, we’ll focus on the concept of decoupling—what it means and, more importantly, how you can achieve it in your codebase.

## The Pitfalls of Coupled Code
> Coupling is the enemy of change, because it link together things that must change in parallel.

When code is tightly coupled, changes in one part of the system often lead to unintended consequences or require modifications in other parts. In the book _The Pragmatic Programmer_, David Thomas and Andrew Hunt identifie some of symptoms of coupling :
- Wacky dependencies between unrelated modules or libraries
- "Simple" changes to one module that propagate through unrelated modules in the system or break stuff elsewhere in the system
- Developers who are afraid to change code because they aren't sure what might be affected
- Meetings where everyone has to attend because no one is sure who will be affected by the change

## How create decoupled code
### Tell, Don't ask
> One of the fundamental principles of object-oriented design is to combine data and behavior, so that the basic elements of our system (objects) combine both together. 

It suggests that instead of asking an object about its internal information and making decisions externally, we should directly tell the object what we want it to do. This promotes a clear separation of responsibilities and helps make code easier to understand and modify.

```java
class BankAccount {
    private double balance;

    public BankAccount(double balance) {
        this.balance = balance;
    }

    public double getBalance() {
        return balance;
    }

    public void setBalance(double balance) {
        this.balance = balance;
    }
}

class BankService {
    public void withdraw(BankAccount account, double amount) {
        if (account.getBalance() >= amount) {
            account.setBalance(account.getBalance() - amount);
        } else {
            System.out.println("Insufficient balance");
        }
    }
}
```

In the *Tell, Don't Ask* approach, we *encapsulate* the behavior inside the BankAccount object, improving cohesion and reducing external decision-making.

```java
class BankAccount {
    private double balance;

    public BankAccount(double balance) {
        this.balance = balance;
    }

    // Tell the BankAccount what to do
    public void withdraw(double amount) {
        if (balance >= amount) {
            balance -= amount;
        } else {
            System.out.println("Insufficient balance");
        }
    }

    // You could also add other behaviors, such as deposit()
    public void deposit(double amount) {
        balance += amount;
    }
}

public static void main(String[] args) {
    BankAccount account = new BankAccount(1000);
    account.withdraw(500);  // Telling the account to withdraw
    account.withdraw(600);  // Insufficient balance
}
```

In the *Tell, Don't Ask* approach, the `BankAccount` class is responsible for knowing its own state and deciding what to do with it. This keeps the logic encapsulated and the object in control of its own behavior.
So by adding business logic in the domain, the *Tell, Don't Ask* principle is closely related to the concept of an [anemic domain model](https://martinfowler.com/bliki/AnemicDomainModel.html). The following [Kata](https://kata-log.rocks/tell-dont-ask-kata) introduces both concepts.

### Align with SOLID
Decoupling aligns closely with several foundational principles of software design, such as SOLID, DRY, and YAGNI, by promoting maintainability, flexibility, and clarity in a codebase. Here’s how decoupling relates to these principles:

> A module should be responsible to one, and only one, actor.

Decoupling ensures that each module or class has a single reason to change by isolating responsibilities. When modules are independent, changes in one area of the codebase are less likely to ripple through others, supporting SRP.

> Software entities (classes, modules, functions, etc.) should be open for extension, but closed for modification

Decoupling also supports OCP by making components open for extension but closed for modification. When code is decoupled, you can add new features by extending existing components rather than altering them directly. For example, a payment system can support additional payment gateways by implementing an interface, without modifying the core logic.

To achieve decoupling and SOLI principles we use DIP, as it advocates depending on abstractions rather than concrete implementations. This makes systems more flexible and easier to extend or refactor.

## Broader Perspectives
In the last section, we focused on concrete examples of code and how principles like Tell, Don't Ask can reduce coupling at the implementation level. Now, let’s take an abstraction perspective, exploring architectural approaches to avoid coupling in a software system

### Bounded Context
The concept of Bounded Context originates from Domain-Driven Design (DDD) and serves as a powerful tool for reducing coupling by clearly defining the boundaries of a domain and its associated logic. Within a bounded context, all elements—data, logic, and rules—are cohesive and operate independently from other contexts. This separation minimizes dependencies and promotes clarity across the entire system.

Key Principles of Bounded Context:
- Each bounded context encapsulates a specific domain or subdomain, ensuring that functionality and data relevant to it do not spill into other parts of the system.
- Communication between contexts happens via explicit interfaces, such as APIs or messaging systems.
- Changes within one bounded context should not necessitate changes in others.

### Modular Monolith
In the blog series [Modular Monolith series](https://www.kamilgrzybek.com/blog/series/modular-monolith) the author emphasize the notion of *modularity* in software design. 

> Modular programming is a software design technique that emphasizes separating the functionality of a program into independent, interchangeable modules, such that each contains everything necessary to execute only one aspect of the desired functionality. A module interface expresses the elements that are provided and required by the module. The elements defined in the interface are detectable by other modules. The implementation contains the working code that corresponds to the elements declared in the interface.

Kamil Grzybek outlines three key characteristics of modules:
- They must be independent and interchangeable.
- They must contain everything necessary to provide the desired functionality.
- They must have a well-defined interface.

## Benefits decoupled code
So, decoupled code offers numerous benefits that enhance the overall quality of a software system.

### Scalability
By dividing the system into independent contexts, teams can work on different parts of the system without stepping on each other's toes. Then each modules could be scaled independently based on their specific load requirements, optimizing resource usage.

### Maintainability
Decoupled components ensure that changes in one part of the system do not cascade to unrelated areas. This reduces the risk of introducing bugs and makes the system easier to maintain.

### Reusability
Decoupled modules, with their well-defined boundaries and interfaces, are inherently more reusable, enabling their seamless integration across various parts of an application or even in entirely different projects.


## Further reader
- https://martinfowler.com/bliki/TellDontAsk.html
- https://martinfowler.com/bliki/AnemicDomainModel.html
- https://www.kamilgrzybek.com/blog/posts/modular-monolith-primer
- [Low Coupling and High Cohesion](../low_coupling_high_cohesion)