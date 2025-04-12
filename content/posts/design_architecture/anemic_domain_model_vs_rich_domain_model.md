+++
title = "Anemic Domain Model vs Rich Domain Model"
series = ["Design Layered Architecture"]
date = "2025-03-23"
+++

In typical Spring project we develop our application with three distinct responsibilities classes
- @Controller which handle JSON request and response
- @Service manage business logic
- @Repository handle the database access
- And finally we have your entity classes mapped with ORM framework and annotated with @Entity

While this structure provides clean separation of concerns, it often leads to the creation of an [Anemic Domain Model](https://martinfowler.com/bliki/AnemicDomainModel.html). In this model, JPA entities serve as mere data containers—essentially just getters and setters—without any domain behavior. This violates core object-oriented principles, where **classes are expected to encapsulate both state (data) and behavior (logic)**.

In such a traditional (and often poorly designed) Spring application:
- Entities are reduced to passive data structures that simply reflect the database schema and relationships.
- Business logic is offloaded entirely to `*Service` classes, detaching behavior from the domain model itself.

## Anemic Domain Model

```java
@Entity
public class UserAccount {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String email;
    private String firstName;
    private String lastName;

    // Getters and setters only – no behavior
}
```

```java
@Service
public class UserAccountService {

    public void createUser(UserAccount user) {
        if (!isValidEmail(user.getEmail())) {
            throw new IllegalArgumentException("Invalid email");
        }

        // Transform last name to uppercase before saving
        user.setLastName(user.getLastName().toUpperCase());

        // Save user to database...
    }

    private boolean isValidEmail(String email) {
        return email != null && email.matches("^[\\w-.]+@([\\w-]+\\.)+[\\w-]{2,4}$");
    }
}
```

## Rich Domain Model
```java
@Entity
public class UserAccount {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String email;
    private String firstName;
    private String lastName;

    protected UserAccount() {
        // For JPA
    }

    public UserAccount(String email, String firstName, String lastName) {
        if (!isValidEmail(email)) {
            throw new IllegalArgumentException("Invalid email format");
        }
        this.email = email;
        this.firstName = firstName;
        this.lastName = lastName.toUpperCase(); // Enforce uppercase on creation
    }

    public void updateLastName(String newLastName) {
        this.lastName = newLastName.toUpperCase(); // Always store in uppercase
    }

    private boolean isValidEmail(String email) {
        return email != null && email.matches("^[\\w-.]+@([\\w-]+\\.)+[\\w-]{2,4}$");
    }
```

Business logic (email validation, formatting) is embedded in the domain class, not in the service.

### What happen when business logic grows
You don’t want `UserAccount` to turn into a 500-line monster. Here's how you'd handle it the DDD (Domain-Driven Design) way.

For example, let’s say you want to send a welcome email when a new account is created. That’s not the entity’s job — it's not part of the domain logic that UserAccount itself should own.  
Instead, we delegate this responsibility to the Service Layer, which orchestrates the domain and integrates external concerns like email.


```java
@Service
public class UserRegistrationAppService {

    @Autowired private final UserAccountRepository userRepository;
    @Autowired private final EmailSender emailSender;

    public void register(UserAccountDTO accountDTO) {
        UserAccount user = new UserAccount(accountDTO.getEmail(), accountDTO.getFirstName(), accountDTO.getLastName());
        
        userRepository.save(user); // Persist the domain object

        emailSender.sendWelcomeEmail(user.getEmail()); // Integrate with infrastructure
    }
}
```

## Transaction Script and Domain Model
Chapter 10 and 11 of *Pattern of Enterprise Architecture*

The first solution with anemic domain model is tied up *[Transaction Script pattern](https://martinfowler.com/eaaCatalog/transactionScript.html)*.

**when to use it**
> The glory of Transaction Script is its simplicity. Organizing logic this way is natural for applications with only a small amount of logic, and it involves very little overhead either in performance or in understanding.

In other words, if your application is small and the business logic is straightforward, it’s perfectly fine to have a `Service.createUser()` method that handles everything: validation, persistence, and notifications 

As complexity increases — multiple rules, cross-cutting concerns, workflows — Transaction Script starts to break down:
- Business logic gets scattered across procedural service methods.
- Invariants are not enforced consistently.
- It’s harder to test and reason about domain behavior.

> It's difficult to keep a coherent design with *Transaction Script* one things get that complicated, which is why objects bigots like me prefer using a *Domain Model* is these circumstances (page 115)

## Further reading
- https://www.devdiaries.net/blog/Java-Anemic-Domain-Model-Rich-Domain-Model/
- https://dzone.com/articles/anemic-domain-model-in-typical-spring-projects-1
- https://stackoverflow.com/questions/16139941/transaction-script-is-antipattern