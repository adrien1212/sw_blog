+++
title = "The Minimal Service Layer Pattern"
series = ["Design Layered Architecture"]
date = "2023-08-20"
+++

## Problem
Different people or system need to perform the same operation, and defining an high-impact Service Layer could reduce the reach of requirement.
A high-impact Service Layer goes beyond just handling business logic such as filtering authentication and authorization.

Consider the following User Stories

> As a user
> I want to access to my order
> (So that I can track my purchase and stay informed about its status)

> As an admin
> I can access to user's order details
> (So that I can manage orders, assist with customer inquiries, and ensure smooth order processing.)

> As a third part microservice
> I can access to the user's order details

To implement a secure and efficient solution, the Service should be designed to handle both authentication and authorization responsibilities. This ensures that authenticated users can access only the specific information they're permitted to view.

For instance, considering the first user story, where a user wants to access their order details, the system must ensure that the user is authenticated and authorized to view only their own order information. This can be achieved by leveraging the Authentication object in Java, using the Spring framework.

```java
@Service
class Service {
	public List<Order> getUserOrders() {
		Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
        String userId = authentication.getId();

        // Fetch the order details for the authenticated user only
        return orderRepository.getOrderByUserId(userId);
	}
}
```

By retrieving information directly from the authenticated user, we ensure that the returned data is specific to that user. However, this approach has limitations:
- What happens when an administrator needs to access a user's orders (user story 2)? The `getUserOrder` method will only return the data associated with the authenticated administrator, not the intended user's information.
- What happens when another microservice needs access (user story 3)? It won't work either, as the third-party microservice isn't registered as an authenticated user.

## Solution

### Delegate to Higher level
The ideal place to handle security is typically at a higher level, usually in the web or API layer. This approach is more efficient, secure, and follows the principle of separation of concerns better.

```java
@GetMapping("order/user/{userId}")
public ResponseEntity<List<Order>> getUserOrders(Long userId) {
	Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
	String authenticatedUserId = authentication.getUserId();

	if(userId != authenticatedUserId || !authentication.getRole().equals("ADMIN")) {
		return ResponseEntity.status(HttpStatus.FORBIDDEN).build();
	}

	List<Order> orders = orderService.getUserOrders(authenticatedUserId);

	return ResponseEntity.ok(orders);
}
```

```java
public List<Order> getUserOrders(Long userId) {
	return orderRepository.getOrderByUserId(userId);
}
```

## Delegate to another Service 
According the [this amazing article](https://herbertograca.com/2017/11/16/explicit-architecture-01-ddd-hexagonal-onion-clean-cqrs-how-i-put-it-all-together/#application-layer)
> Bounded contexts like Authorization and/or Authentication should be seen as external tools for which we create an adapter and hide behind some kind of port.

```java
public Order getUserOrder(String token) {
	if (!authService.verifyUserOrAdminToken(token)) {
		throw new UnauthorizedException("Invalid token");
	}

	...

	return order;
}
```

## The trade-off
By applying security checks in the Service Layer we ensure that even if someone bypasses the Presentation Layer, proper authorization is enforced before any sensitive data is processed. The Service Layer could resides in one library, and being used by different Presentation Layers. And we can't rely on the fact that specific Presentation layers perform necessary validation. 

However, if your application requires more complex access control scenarios, such as administrators accessing other users' data or external services needing specific permissions, relying solely on the Service Layer might not be sufficient. In such cases, a more nuanced approach is required :
- Presentation Layer filtering
- Third party Service

## Conclusion
I called it "The Minimal Service Layer Pattern" because the Service Layer only know the minimum information.

> It encapsulates the application's business logic, controlling transactions and coor-dinating responses in the implementation of its operations - [Martin Fowler](https://martinfowler.com/eaaCatalog/serviceLayer.html)

And delegates all the superfluous stuff