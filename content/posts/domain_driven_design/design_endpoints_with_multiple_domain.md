+++
title = "Designing the GET /customers/{id}/orders Endpoint"
series = ["Domain Driven Design"]
date = "2025-03-23"
+++

Recently, while reviewing the talk [The Modular Monolith – a Practical Alternative to Microservices](https://youtu.be/nuHMlA3iLjY) by Victor Rentea, I found myself pondering a seemingly simple question: where should we implement the logic for `GET /customers/{id}/orders` ?

In this article, I’ll walk you through three possible design approaches, the trade-offs of each, and why the final choice not only respects encapsulation but also scales well within a monolithic architecture. The discussion was shaped by insights from Victor himself, and it helped clarify a lot about practical modular design.

## Why we're interested in this endpoint?
Some of view, will say, "Adrien, why focus so much on this one endpoint?". Give me 1 minute to show how many questions this endpoint raises.

Coding the endpoint `GET /order/{id}` is pretty straightforward. Victor Rentea give us [the code](https://github.com/victorrentea/spring-modulith/blob/devoxx-uk-24/order/src/main/java/victor/training/modulith/order/impl/OrderRestApi.java)
- The owner of the endpoint is `OrderRestApi` – the API layer of the Order module.
- The business logic is delegated to `OrderService`.
- `OrderService` then calls `OrderRepository` to access the data.

Now, please answers to this question **"How get all orders of a specific user ?"**
- Where does the API layer belongs ? `OrderRestAPI` or `CustomerRestAPI`
- Where does the business logic really belong? `OrderService` or `CustomerService`
- How do we maintain clean boundaries between modules?

## How get all orders of a specific user ?
One possible approach is to design a `GET /customers/{id}/orders` endpoint. In this section, we’ll explore three different implementation strategies, each with its own set of advantages and trade-offs.

### Hypothesis 1: Return Orders from CustomerService
```java
public class Customer {
    List<Order> orders;
}
```

```java
public class CustomerRestAPI {
    private final CustomerService customerService;

    @GetMapping("customers/{id}/orders")
    public ResponseEntity<List<Order>> getOrdersByCustomerId(@PathVariable Integer id) {
        List<Order> orders = customerService.getOrdersByCustomerId(id);
        return ResponseEntity.ok(orders);
    }
}

public class CustomerService {
    public List<Order> getOrderByCustomerId(Integer customerId) {
        return customerRepository.findById(customerId).getOrders();
    }
}
```

However, this approach breaks encapsulation, since the customer package would now depend on *Domain Entity* of the order package.

![image](../images/design_endpoints_with_multiple_domain/hypothesis1.png)

### Hypothesis 2: Implement getOrderByCustomerId in OrderService
```java
public class Customer { 
    // List<Order> orders; no longer reference Order 
}

public class Order {  private Integer customerId; } 
```

```java
public class OrderService {
    public List<Order> getOrderByCustomerId(Integer customerId) {
        return orderRepository.findByCustomerId(customerId);
    }
}
```

This design is cleaner in terms of boundaries, as the Order package only needs the customerId and doesn’t depend on the Customer package.

However, the CustomerRestApi would now need to depend on both CustomerService and OrderService

```java
@RestController
public class CustomerRestApi {
    private final CustomerService customerService;
    private final OrderService orderService;

    @GetMapping("customers/{customerId}")
    public Customer getCustomerById(@PathVariable Integer customerId) {
        return customerService.findById(customerId);
    }

    @GetMapping("customers/{customerId}/orders")
    public List<Order> getOrdersByCustomerId(@PathVariable Integer customerId) {
        return orderService.getOrderByCustomerId(customerId);
    }
}
```

In this architecture, the customer package depends on the order package—but only through its public API layer (aka `OrderService`), not the domain entities directly.

![image](../images/design_endpoints_with_multiple_domain/hypothesis2.png)

### Hypothesis 3: Two Bounded Contexts — Only IDs Are Shared
This is aligned with your domain-driven design approach. [Order only knows the customerId (see line 25)](https://github.com/victorrentea/spring-modulith/blob/master/src/main/java/victor/training/modulith/order/impl/Order.java#L25), and similarly, Customer can have a `List<Integer> orderIds`. In this case, the CustomerService could return the order IDs

```java
class Customer {
    List<Integer> ordersId
}
```

```java
@RestController
public class CustomerRestApi {
    private final CustomerService customerService;

    @GetMapping("customers/{customerId}/orders")
    public List<Integer> getOrdersByCustomerId(@PathVariable Integer customerId) {
        return customerService.getOrderByCustomerId(customerId);
    }
}

public class CustomerService {
    public List<Integer> getOrderByCustomerId(Integer customerId) {
        return customerRepository.findById(customerId).getOrderIds();
    }
}
```
With this approach, the packages are now completely decoupled. However, this comes at the cost of additional complexity on the client side. The frontend must iterate over each orderId to fetch the corresponding order details individually, which can result in multiple network calls

![image](../images/design_endpoints_with_multiple_domain/hypothesis3.png)

## Better approach
### Heuristic
As a general rule, domain entities should not directly reference each other[^1]. Instead:
- The `Order` entity should hold a `customerId`.
- The `Customer` entity could maintain a `List<Integer> orderIds`.

However, in this specific use case, storing orderIds in the Customer domain doesn’t provide meaningful value—it introduces unnecessary duplication and potential for inconsistency.

### So how design GET /customers/{id}/orders ?
A cleaner and more decoupled alternative is to invert the query. Rather than asking the Customer module for its orders, let the Order module expose a query endpoint:

`GET /order?customerId=123`

This shifts the responsibility to the Order bounded context, which owns the data and logic related to orders, and is therefore the most appropriate place to perform filtering by customerId.

[^1]: https://enterprisecraftsmanship.com/posts/link-to-an-aggregate-reference-or-id/
I agree with this article, but in your case with JPA and Domain Entity linked (maybe considered a “bad practice”)

```java
@RestController
@RequestMapping("/orders")
public class OrderRestApi {
    private final OrderService orderService

    @GetMapping
    public List<Order> getOrdersByCustomerId(@RequestParam Integer customerId) {
        return orderService.getOrdersByCustomerId(customerId);
    }
}

@Service
public class OrderService {
    private final OrderRepository orderRepository;

    public List<Order> getOrdersByCustomerId(Integer customerId) {
        return orderRepository.findByCustomerId(customerId);
    }
}
```