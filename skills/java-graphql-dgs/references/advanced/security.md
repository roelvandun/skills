# DGS Security Reference

## Integration with Spring Security

DGS integrates seamlessly with Spring Security. Once Spring Security is configured, apply `@Secured` directly to data fetcher methods — the same way you'd use it on a Spring MVC controller.

## Using `@Secured`

```java
@DgsComponent
public class SecurityExampleFetcher {

    @DgsQuery
    public String hello() {
        return "Hello to everyone";
    }

    @Secured("ROLE_ADMIN")
    @DgsQuery
    public String adminOnly() {
        return "Hello to admins only";
    }
}
```

Users without the `ROLE_ADMIN` role receive a GraphQL error with `PERMISSION_DENIED`.

## Using `@PreAuthorize`

```java
@DgsComponent
public class SecureFetcher {

    @PreAuthorize("hasRole('USER')")
    @DgsQuery
    public List<Order> myOrders(DataFetchingEnvironment dfe) {
        // Only accessible to authenticated users with ROLE_USER
        return orderService.findForCurrentUser();
    }

    @PreAuthorize("#username == authentication.principal.username")
    @DgsQuery
    public Profile profile(@InputArgument String username) {
        return profileService.findByUsername(username);
    }
}
```

## Spring Security Configuration

Standard Spring Security setup — DGS requires no special security configuration:

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity  // Required for @Secured and @PreAuthorize
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/graphql").authenticated()
                .anyRequest().permitAll()
            )
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
            .build();
    }
}
```

## Accessing the Principal in Data Fetchers

```java
@DgsQuery
public UserProfile me() {
    Authentication auth = SecurityContextHolder.getContext().getAuthentication();
    String username = auth.getName();
    return userService.findByUsername(username);
}
```

Or inject via Spring:

```java
@DgsQuery
public UserProfile me(@AuthenticationPrincipal UserDetails user) {
    return userService.findByUsername(user.getUsername());
}
```

## Note on Virtual Threads

When `dgs.graphql.virtualthreads.enabled=true`, data fetchers run on different threads. Spring Security's `SecurityContext` uses `ThreadLocal` by default — ensure you configure context propagation for virtual threads:

```java
SecurityContextHolder.setStrategyName(SecurityContextHolder.MODE_INHERITABLETHREADLOCAL);
```
