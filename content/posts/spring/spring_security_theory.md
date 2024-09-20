+++
title = "Spring Security Theory"
series = ["Spring Security"]
weight = 10
date = "2024-09-04"
+++

While Spring Security is a powerful framework for securing Java applications, it can be challenging to grasp and implement effectively. Perhaps the most significant hurdle for developers is the lack of a clear understanding of its fundamental concepts and core principles. In this article we will not code, I just show you of Spring Security works through diagrams.

![Spring Security](../images/spring_theory/spring_security_theory.png)

![Spring Security Sequence diagram](../images/spring_theory/spring_security_theory_sequence.png)

## Architecture
The user want access to `GET /privatepage`

### SecurityFilterChain
A series of Spring Security filters intercept each request (so `GET /privatepage`) & work together to identify if Authentication is required or not. If one filter fail (e.g. `AbstractAuthenticationProcessingFilter`) the user cannot access to the ressource.

The `AbstractAuthenticationProcessingFilter` is used as a base Filter for authenticating a user’s credentials. Several implementation exists as  `OAuth2LoginAuthenticationFilter` or `UsernamePasswordAuthenticationFilter`.

![Security Filter Chain](../images/spring_theory/spring_security_theory_sf.png)

And when the user submits their credentials, the `AbstractAuthenticationProcessingFilter` creates an `Authentication` object. The type of Authentication created depends on the subclass of `AbstractAuthenticationProcessingFilter`. For example, `UsernamePasswordAuthenticationFilter` creates a `UsernamePasswordAuthenticationToken` from a username and password that are submitted.

### AuthenticationManager
Next, the `Authentication` is passed into the `AuthenticationManager` to be authenticated. The `AuthenticationManager` delegates to a List of `AuthenticationProvider` instances the opportunity to indicate that authentication should be successful, fail, or indicate it cannot make a decision.

![AuthenticationManager](../images/spring_theory/spring_security_theory_manager.png)

### AuthenticationProvider
A common implementation of `AuthenticationProvider` is `DAOAuthenticationProvider` that uses a `UserDetailsService` and `PasswordEncoder` to authenticate a username and password. The `UserDetailsService` define an method to retrieve the user from the database for example.

![AuthenticationProvider](../images/spring_theory/spring_security_theory_provider.png)

## The user is authenticated
When authentication is successful, the `Authentication` that is returned is of type `UsernamePasswordAuthenticationToken` and has a principal that is the `UserDetails` returned by the configured `UserDetailsService`. Ultimately, the returned `UsernamePasswordAuthenticationToken` is set on the `SecurityContextHolder` by the authentication Filter.

![SecurityContextHolder](../images/spring_theory/spring_security_theory_sch.png)

Now the user can access to the request ressource `GET /privatepage` and get the result of the page
## Further reading
- https://docs.spring.io/spring-security/reference/servlet/authentication/architecture.html
- [Spring Security explained with no code](https://youtu.be/n6pXL9M0ljM)