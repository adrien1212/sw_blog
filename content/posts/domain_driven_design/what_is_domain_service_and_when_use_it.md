+++
title = "What is a domain service and when should you use it?"
series = ["Domain Driven Design"]
date = "2026-06-05"
+++

When you start using Domain-Driven Design (DDD) in your Java projects, you will quickly encounter some core building blocks: Entities, Value Objects, Aggregates, and finally, *Domain Services*.

But what exactly is a Domain Service, and when should you use one?

## From definition to a concrete example

> A service in the domain is a stateless operation that performs a domain-specific task. A good sign that you should create a service in the domain model is when the operation feels out of place as a method on an Aggregate or a Value Object. [^1]

The following operation, `credit(double amount)`, naturally belongs to a single Entity, `Account`:

```java
class Account {
    private double balance;

    public void credit(double amount) {
        if (amount < 0) {
            throw new IllegalArgumentException("Amount must be positive");
        }
        this.balance += amount;
    }
}
```

Now imagine a simple banking system where you can transfer money between accounts. Where should the business logic go?

### Hypothesis 1: the Account entity

```java
class Account {
    private double balance;

    public void transfer(Account dest, double amount) {
        if (this.balance < amount) {
            throw new InsufficientFunds();
        }
        this.balance -= amount;
        dest.balance += amount;
    }
}
```

A few things can go wrong with this design:
- It mixes responsibilities. `Account` is now doing both its own balance rules and the transfer workflow between two accounts.
- It can break encapsulation. `dest.solde += amount` directly changes another account’s internal state, which is usually a smell.

### Hypothesis 2 : place in Application Service

```java
class TransferApplicationService {
    private final AccountRepository accountRepository;

    @Transactional
    public void transfer(String senderIban, String recipientIban, double amount) {
        if (amount <= 0) {
            throw new IllegalArgumentException("Amount must be positive");
        }

        Account sender = accountRepository.findByIban(senderIban);
        Account recipient = accountRepository.findByIban(recipientIban);

        if (sender.getBalance() < amount) {
            throw new InsufficientFunds();
        }

        sender.debit(amount);
        recipient.credit(amount);

        accountRepository.save(sender);
        accountRepository.save(recipient);
    }
}
```

A few things can go wrong with this design:
- The application service becomes too business-heavy. It is now doing validation, balance checks, and money movement logic instead of just orchestrating the use case.
- The transfer rule is duplicated outside the domain model. If another use case also transfers money, you may copy the same logic again.
- The business rule is less reusable. The logic lives in one service method instead of inside the Account model or a domain service.

Another indicator is how easy it is to unit test this code. Here, we need to mock dependencies to test the transfer.

### The right : use Domain Service
A better approach is :
- keep `Account` responsible for its own state and rules
- keep `TransferApplicationService` responsible for orchestration
- use a *Domain Service* to coordinate the transfer between accounts

```java
/** Domain Service */
class TransferService {
    public void transfer(Account sender, Account recipient, double amount) {
        if (amount <= 0) {
            throw new IllegalArgumentException("Amount must be positive");
        }

        if (sender.getBalance() < amount) {
            throw new InsufficientFunds();
        }

        sender.debit(amount);
        recipient.credit(amount);
    }
}
```


```java
class TransferApplicationService {
    private final AccountRepository accountRepository;
    private final TransferService transferService;

    @Transactional
    public void transfer(String senderIban, String recipientIban, double amount) {
        Account sender = accountRepository.findByIban(senderIban);
        Account recipient = accountRepository.findByIban(recipientIban);

        transferService.transfer(sender, recipient, amount);

        accountRepository.save(sender);
        accountRepository.save(recipient);
    }
}
```

This design is good because it separates responsibilities clearly.
- `TransferApplicationService` handles orchestration: it loads data, calls the transfer logic, and saves the result.
- `TransferService` holds the business rule for transferring money.
- `Account` keeps its own state and low-level behavior like `debit` and `credit`.

Also creating unit test is easy on class `Account` and `TransfertService`, no mock required.
```java
@Test
void transfers_money_between_accounts() {
    Account sender = new Account(100.0);
    Account recipient = new Account(20.0);

    TransferService transferService = new TransferService();

    transferService.transfer(sender, recipient, 30.0);

    assertEquals(70.0, sender.getBalance());
    assertEquals(50.0, recipient.getBalance());
}
```

## Another example 
The transfer example shows a clear case where the business rule does not belong to a single entity. But not every rule needs a domain service.

Imagine we need to validate an order before confirming it. One of the business rules is simple: the client must be at least 18 years old.

A domain service can still centralize this rule and keep the domain model clear. The application service can load the order and the customer, then ask the domain service to validate whether the order can be placed.

```java
/* Domain Service */
class OrderValidationService {
    public void validateCustomerAge(Customer customer) {
        if (customer.getAge() < 18) {
            throw new UnderAgeCustomerException();
        }
    }
}
```

```java
class OrderApplicationService {
    private final OrderRepository orderRepository;
    private final CustomerRepository customerRepository;
    private final OrderValidationService orderValidationService;

    @Transactional
    public void placeOrder(String orderId, String customerId) {
        Order order = orderRepository.findById(orderId);
        Customer customer = customerRepository.findById(customerId);

        orderValidationService.validateCustomerAge(customer);

        order.confirm();

        orderRepository.save(order);
    }
}
```

> Domain services are coordinators that allow higher-level functionality between many smaller parts. They include things like `OrderProcessor`, `ProductFinder`, and `FundsTransferService`. Since domain services are first-class citizens of our domain model, their names and usage should be part of the Ubiquitous Language. Their meaning and responsibilities should make sense to stakeholders or domain experts. [^2]

### But another design possible
If the rule is only “customer must be 18+ to place this order,” `Order.validateFor(Customer)` is perfectly reasonable. A domain service becomes more useful when the validation becomes more complex.

So, for a simple use case, several designs are possible:

```java
class Order {
    public void validateFor(Customer customer) {
        if (customer.getAge() < 18) {
            throw new UnderAgeCustomerException();
        }
    }
}
```

## Conclusion

A domain service is useful when a business rule does not naturally belong to a single entity or value object. It helps keep the domain model clear by moving coordination logic into the domain layer.

At the same time, not every rule needs a domain service. If the behavior clearly belongs to one entity, it is often better to keep it there. The goal is to make the model simple, expressive, and easy to evolve.

[^1]: Implementing Domain-Driven Design - Vaughn Vernon
[^2]: https://lostechies.com/jimmybogard/2008/08/21/services-in-domain-driven-design/
