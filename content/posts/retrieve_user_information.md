+++
title = "Retrieve User Information in Spring Security"
series = ["Spring Security"]
date = "2023-08-31"
+++

Spring Security provides several ways to retrieve information about the authenticated user. Let's explore how Spring Security stores authenticated user details and the methods to access this information.

## Storing authenticated users
Spring Security uses the SecurityContextHolder to store authentication information. This is a thread-local storage mechanism, meaning each thread (including the one processing an HTTP request) has its own SecurityContext. When a user is authenticated, Spring Security creates an Authentication object containing the principal (i.g. user details), credentials, and granted authorities. This Authentication object is then stored in the SecurityContext.

> The SecurityContextHolder is where Spring Security stores the details of who is authenticated.

![Security Context Holder](https://docs.spring.io/spring-security/reference/_images/servlet/authentication/architecture/securitycontextholder.png)

Most authentication mechanisms withing Spring Security return an instance of `UserDetails` as implementation of `Principal`. 

> Most of the time this can be cast into a UserDetails object. UserDetails is a central interface in Spring Security. It represents a principal, but in an extensible and application-specific way. Think of UserDetails as the adapter between your own user database and what Spring Security needs inside the SecurityContextHolder.

By this way, we can access to the authenticated user from the application.

## Accessing Authenticated user information

### Using SecurityContextHolder
 The most straightforward programmatic approach is to use the SecurityContextHolder

```java
Object principal = SecurityContextHolder.getContext().getAuthentication().getPrincipal();
String username = ((UserDetails)principal).getUsername();
```

An improvement over the first code snippet is to use the authentication object

```java
Object authentication = SecurityContextHolder.getContext().getAuthentication()
String username = authentication.getName();
```

### Injecting Authentication Object

You can also access to the user's name and roles directly by injecting `Authentication` object, and the framework will correctly resolve it

```java
@GetMapping("/user")
public String currentUser(Authentication authentication) {
    return authentication.getName();
}
```

We can define the principal or the authentication directly as a method argument, 

### Using @AuthenticationPrincipal

Spring Security provides the `@AuthenticationPrincipal` annotation, which can be used to retrieve the UserDetails object directly in your controller method. It automatically resolves the `UserDetails` object from the `Authentication` object and injects it into the method parameter.

```java
@GetMapping("/user")
public String currentUser(@AuthenticationPrincipal UserDetails userDetails) {
    return userDetails.getUsername();
}
```

This alternative allow you tu use a custom `UserDetails` implementation and define more attributes.

```java
public class CustomUserDetails implements UserDetails {
    private String username;
    private String password;
    private String email;
    private String fullName;
    // ... other UserDetails methods

    public String getEmail() {
        return email;
    }

    public String getFullName() {
        return fullName;
    }
}

@GetMapping("/user")
public String currentUser(@AuthenticationPrincipal CustomUserDetails customUser) {
    return customUser.getEmail() + " " + customUser.getFullName();
}
```

## User only to access their own data
In a monolithic application, each component can easily access the SecurityContextHolder through an in-memory call. This simplifies the process of authenticating and authorizing users for specific operations.

> As a user
> I want to access to my order
> (So that I can track my purchase and stay informed about its status)

![Monolith](../images/spring_authentication/monolith.png)

```java
@GetMapping("/order/user/{username}")
public ResponseEntity<List<OrderDTO>> getOrdersByUser(@PathVariable String username, 
                                                      @AuthenticationPrincipal String pUsername) {
    if (!pUsername.equals(username)) {
        return ResponseEntity.status(HttpStatus.UNAUTHORIZED).build();
    }

    // The logged user is the user that made the request
    return ResponseEntity.ok(orderService.getOrdersByUser(username));
}
```

We can also Spring Security Expression-Based Access Control instead of manual checking. The following `@PreAuthorize` annotation ensures that the method can only be executed if the username in the path variable matches the name of the currently authenticated user.

```java
@GetMapping("/order/user/{username}")
@PreAuthorize("#username == authentication.name")
public ResponseEntity<List<OrderDTO>> getOrdersByUser(@PathVariable String username) {
    return ResponseEntity.ok(orderService.getOrdersByUser(username));
}
```

## Further reading

- https://www.marcobehler.com/guides/spring-security#_authentication_with_spring_security
- https://docs.spring.io/spring-security/site/docs/3.0.x/reference/el-access.html