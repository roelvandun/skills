# DGS Operation Caching Reference

## Overview

Before executing a GraphQL operation, graphql-java must **parse** and **validate** the query string. These steps are CPU-intensive and repeated for every request with the same query text.

DGS supports injecting a `PreparsedDocumentProvider` to cache the parsed document, eliminating redundant parse/validate cycles.

## Caffeine Cache Implementation

```java
@Component
public class CachingPreparsedDocumentProvider implements PreparsedDocumentProvider {

    private final Cache<String, PreparsedDocumentEntry> cache = Caffeine.newBuilder()
        .maximumSize(2500)
        .expireAfterAccess(Duration.ofHours(1))
        .build();

    @Override
    public PreparsedDocumentEntry getDocument(
            ExecutionInput executionInput,
            Function<ExecutionInput, PreparsedDocumentEntry> parseAndValidateFunction) {
        // Cache key is the query string
        return cache.get(
            executionInput.getQuery(),
            query -> parseAndValidateFunction.apply(executionInput)
        );
    }
}
```

Add Caffeine dependency:

```xml
<dependency>
    <groupId>com.github.ben-manes.caffeine</groupId>
    <artifactId>caffeine</artifactId>
</dependency>
```

## Using Operation Variables (Required for Effective Caching)

The cache key is the **query string**. Operations with inline literals create unique cache entries per call:

```graphql
# BAD — unique cache entry for every different ID
{ person(id: "123") { firstName } }
{ person(id: "456") { firstName } }
```

Use variables instead:

```graphql
# GOOD — one cache entry for all personId values
query GetPerson($personId: String!) {
  person(id: $personId) { firstName }
}
```

Pass `$personId` as a variable in the request, not embedded in the query string.

## Benefits

- Eliminates parse/validation overhead for frequently used operations
- Particularly beneficial for mobile/IoT clients that send the same queries repeatedly
- Works well with persisted queries (APQ) patterns

## Via `@Bean` Method

```java
@Configuration
public class DgsConfig {

    @Bean
    public PreparsedDocumentProvider preparsedDocumentProvider() {
        Cache<String, PreparsedDocumentEntry> cache = Caffeine.newBuilder()
            .maximumSize(2500)
            .expireAfterAccess(Duration.ofHours(1))
            .build();
        return (input, parseAndValidate) ->
            cache.get(input.getQuery(), q -> parseAndValidate.apply(input));
    }
}
```
