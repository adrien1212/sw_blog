+++
title = "Spring Security Minimum"
series = ["Spring Security"]
weight = 15
date = "2023-09-11"
+++
This article and the following code represent the minimal configuration to add security in your Spring application. We will use the latest article [Spring Security Theory](../spring_security_theory) where we have described the Spring Security Architecture with these main component : Security Filter Chain, AuthenticationManager, AuthenticationProvider and the Security Context Holder. It is therefore important to read this last article in order to understand the following code.

![Spring Security](../images/spring_theory/spring_security_theory.png)


## Maven dependencies
The code is available on github [https://github.com/Adrien-Courses/spring-security-minimum](https://github.com/Adrien-Courses/spring-security-minimum)

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

## Main component
The `SecurityConfig` class is annotated with `@EnableWebSecurity` to enable Spring Security’s web security support and provide the Spring MVC integration. 
It also exposes severals beans to set some specifics for the web security configuration :
- the SecurityFilterChain
- the AuthenticationProvider
  - the UserDetailService
  - the PasswordEncoder

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    // All code go here
}
```

### Authentication Manager
The `AuthenticationManager` is really just a container for authentication providers, giving a consistent interface to them all. In general, the `AuthenticationManager` passes some sort of `AuthenticationToken` (e.g. `UsernamePasswordAuthenticationToken`) to the each of it's `AuthenticationProviders` and they each inspect it and, if they can use it to authenticate, they return with an indication of "Authenticated", "Unauthenticated", or "Could not authenticate"

```java
@Bean
public AuthenticationManager authenticationManager(List<AuthenticationProvider> authenticationProviders) {
    return new ProviderManager(authenticationProviders);
}
```

### AuthenticationProvider
```java
@Bean
public AuthenticationProvider authenticationProvider() {
    DaoAuthenticationProvider authProvider = new DaoAuthenticationProvider();
    authProvider.setUserDetailsService(userDetailsService());
    authProvider.setPasswordEncoder(passwordEncoder());
    return authProvider;
}
```

The `DAOAuthenticationProvider` needs [UserDetailsService](https://docs.spring.io/spring-security/reference/servlet/authentication/passwords/user-details-service.html) and [PasswordEncoder](https://docs.spring.io/spring-security/reference/servlet/authentication/passwords/password-encoder.html). So we define to more beans. The `UserDetailsService` is an interface with only one method `loadUserByUsername`. In real word application these method call the userRepository to get data from database. But in your exemple in hard-code in-memory the user (instance of `UserDetails`)

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
}

@Bean
public UserDetailsService userDetailsService() {
    return new UserDetailsService() {
        @Override
        public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
            return new UserDetails() {
                @Override
                public Collection<? extends GrantedAuthority> getAuthorities() {
                    return List.of();
                }

                @Override
                public String getPassword() {
                    return passwordEncoder().encode("password");
                }

                @Override
                public String getUsername() {
                    return "user";
                }
            };
        }
    };
}
```

Another way to define `userDetailsService()` is Spring Security’s [InMemoryUserDetailsManager](https://docs.spring.io/spring-security/reference/servlet/authentication/passwords/in-memory.html) that implements UserDetailsService to provide support for username/password based authentication that is stored in memory.

```java
@Bean
public UserDetailsService users() {
	UserDetails user = User.builder()
		.username("user")
		.password(passwordEncoder().encode("password"))
		.roles("USER")
		.build();
	
	return new InMemoryUserDetailsManager(user, admin);
}
```

### SecurityFilterChain
Finally we define our `SecurityFilterChain` by :
- defining the authenticationProvider to use
- requesting all path for the authentication

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http.authorizeHttpRequests(req ->
                    req.requestMatchers("/auth/**").permitAll()
                            .anyRequest().authenticated())
            .authenticationProvider(authenticationProvider())
            .formLogin(Customizer.withDefaults()); // to have a form for the login
    return http.build();
}
```

## Controller
We implement a very simple route to test the application
```java
@RestController
public class AnyController {

    @GetMapping("/private")
    public String mustAuthenticated() {
        return "<h1>You are authenticated</h1>";
    }
}
```

## Tester l'application
When I work with Spring Security I like to set logging level. This give me visibility on what's happening in my application. To do it add the following line in the `application.properties` file
`logging.level.org.springframework.security=TRACE` and now the console show you all the filter trace.

Now, to access the `/private` page we must login `user:password`

## Further reading
- [https://docs.spring.io/spring-security/reference/servlet/authentication/architecture.html](https://docs.spring.io/spring-security/reference/servlet/authentication/architecture.html)
- [https://docs.spring.io/spring-security/reference/servlet/authentication/passwords/storage.html](https://docs.spring.io/spring-security/reference/servlet/authentication/passwords/storage.html)