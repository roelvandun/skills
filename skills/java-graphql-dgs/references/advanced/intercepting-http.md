# DGS HTTP Request/Response Interception Reference

## Overview

Intercept GraphQL HTTP requests and responses using Spring for GraphQL's `WebGraphQlInterceptor`. This is useful for reading headers, adding response extensions, or modifying the context.

## Implementing a `WebGraphQlInterceptor`

```java
@Component
public class AuthHeaderInterceptor implements WebGraphQlInterceptor {

    @Override
    public Mono<WebGraphQlResponse> intercept(WebGraphQlRequest request, Chain chain) {
        // Read a header from the HTTP request
        String authToken = request.getHeaders().getFirst("Authorization");

        // Add to GraphQL context so data fetchers can access it
        request.configureExecutionInput((input, builder) ->
            builder.graphQLContext(Map.of("authToken", authToken)).build()
        );

        // Continue the chain and optionally modify the response
        return chain.next(request).map(response -> {
            // Add response headers or extensions here
            return response;
        });
    }
}
```

## Accessing the Header Value in a Data Fetcher

```java
@DgsQuery
public Show show(@InputArgument String id, DataFetchingEnvironment dfe) {
    String authToken = dfe.getGraphQlContext().get("authToken");
    // Use authToken for authorization
    return showService.findById(id, authToken);
}
```

## Multiple Interceptors

Register multiple `WebGraphQlInterceptor` beans — they chain automatically via the `Chain`. Use `@Order` to control execution order:

```java
@Component
@Order(1)
public class LoggingInterceptor implements WebGraphQlInterceptor { ... }

@Component
@Order(2)
public class AuthInterceptor implements WebGraphQlInterceptor { ... }
```

## Adding Response Extensions

```java
@Override
public Mono<WebGraphQlResponse> intercept(WebGraphQlRequest request, Chain chain) {
    return chain.next(request).map(response -> {
        Map<String, Object> extensions = new HashMap<>(response.getExecutionResult().getExtensions());
        extensions.put("serverVersion", "1.0");
        ExecutionResult modified = response.getExecutionResult().transform(
            builder -> builder.extensions(extensions)
        );
        return response.transform(builder -> builder.executionResult(modified));
    });
}
```

## Notes

- `WebGraphQlInterceptor` is the Spring for GraphQL equivalent of a servlet filter for GraphQL requests
- For subscription transports, the same interceptor applies to the initial HTTP upgrade request
- Do not use DGS-specific context builders for values that must be visible from the HTTP layer; use `GraphQLContext` instead (see `advanced/graphql-context.md`)
