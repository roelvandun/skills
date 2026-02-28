---
name: java-graphql-dgs
description: >
  Guide for creating GraphQL services with Spring Boot using the Netflix DGS framework.
  Use this skill when:
  (1) adding a new GraphQL query, mutation, or subscription to a Spring Boot service,
  (2) designing or updating a DGS data fetcher, data loader, or custom scalar,
  (3) setting up DGS in an existing or new Spring Boot module,
  (4) writing tests for DGS data fetchers or subscriptions,
  (5) working with GraphQL schemas in src/main/resources/schema/,
  (6) configuring federation, security, file uploads, pagination, or instrumentation in DGS.
license: MIT
compatibility: Requires Spring Boot 3 and Java 17. Uses Maven.
metadata:
  author: jumbo-loyalty
  version: "1.0.0"
allowed-tools: Read Write Edit Glob Grep
---

# Netflix DGS Framework Guide

DGS (Domain Graph Service) is a Netflix framework that makes it easy to build GraphQL services on top of Spring Boot. It uses an annotation-based programming model on top of Spring for GraphQL.

## Process

Follow this process when adding or changing a GraphQL feature:

- [ ] Step 1: Define or update the schema in `src/main/resources/schema/*.graphqls`
- [ ] Step 2: Create or update a `@DgsComponent` class for the data fetcher
- [ ] Step 3: Annotate the method with `@DgsQuery`, `@DgsMutation`, or `@DgsSubscription`
- [ ] Step 4: Use `@InputArgument` to bind query arguments
- [ ] Step 5: Write a test using `@EnableDgsTest` and `DgsQueryExecutor`

## Quick Reference

### Maven setup

See [Setup](references/setup.md) for full dependency configuration.

The essential Maven addition (in `dependencyManagement`):

```xml
<dependency>
  <groupId>com.netflix.graphql.dgs</groupId>
  <artifactId>graphql-dgs-platform-dependencies</artifactId>
  <version>10.0.6</version>
  <type>pom</type>
  <scope>import</scope>
</dependency>
```

Plus the starter:

```xml
<dependency>
  <groupId>com.netflix.graphql.dgs</groupId>
  <artifactId>dgs-starter</artifactId>
</dependency>
```

### Schema files

Place all GraphQL schema files in `src/main/resources/schema/` with a `.graphqls` extension. DGS picks them up automatically.

```graphql
type Query {
  user(id: ID!): User
}

type User {
  id: ID!
  email: String!
  name: String
}
```

### Data fetcher

```java
@DgsComponent
@RequiredArgsConstructor
public class UserDataFetcher {

    private final UserService userService;

    @DgsQuery
    public User user(@InputArgument String id) {
        return userService.findById(id);
    }
}
```

### Mutation

```java
@DgsMutation
public User createUser(@InputArgument CreateUserInput input) {
    return userService.create(input);
}
```

Use `input CreateUserInput { ... }` in the schema for mutation arguments.

### Child data fetcher (avoid N+1)

When a field requires a separate query, use a child fetcher:

```java
@DgsData(parentType = "User", field = "loyaltyCard")
public LoyaltyCard loyaltyCard(DgsDataFetchingEnvironment dfe) {
    User user = dfe.getSource();
    return cardService.findByCustomerId(user.getId());
}
```

For collections with N+1 concerns, see [Data Fetchers](references/datafetchers.md#data-loaders).

### Test

```java
@SpringBootTest(classes = {UserDataFetcher.class, UserService.class})
@EnableDgsTest
class UserDataFetcherTest {

    @Autowired
    DgsQueryExecutor dgsQueryExecutor;

    @Test
    void user() {
        String email = dgsQueryExecutor.executeAndExtractJsonPath(
            "{ user(id: \"123\") { email } }",
            "data.user.email"
        );
        assertThat(email).isEqualTo("test@example.com");
    }
}
```

## Reference Files

### Core Topics
- [Setup](references/setup.md) — Maven dependencies, module structure, application properties
- [Data Fetchers](references/datafetchers.md) — Annotations, arguments, child fetchers, data loaders
- [Mutations](references/mutations.md) — `@DgsMutation`, input types, sparse updates
- [Data Loaders](references/data-loaders.md) — `@DgsDataLoader`, `BatchLoader`, `MappedBatchLoader`, `Try`, virtual threads
- [Subscriptions](references/subscriptions.md) — `@DgsSubscription`, WebSocket, SSE, testing
- [Error Handling](references/error-handling.md) — `@ControllerAdvice`, `TypedGraphQLError`, subscription errors
- [Scalars](references/scalars.md) — `@DgsScalar`, extended scalars, `@DgsRuntimeWiring`
- [Code Generation](references/code-generation.md) — Gradle/Maven plugin, DgsConstants, type-safe query API
- [Federation](references/federation.md) — `@key`, `@extends`, `@DgsEntityFetcher`
- [Testing](references/testing.md) — `@EnableDgsTest`, `DgsQueryExecutor`, MockMvc integration
- [Configuration](references/configuration.md) — Full `dgs.graphql.*` property reference
- [Spring GraphQL Integration](references/spring-graphql-integration.md) — DGS/Spring GraphQL relationship, config overlap

### Advanced Topics
- [Context Passing](references/advanced/context-passing.md) — `getSource()`, `DataFetcherResult.localContext`, selection set
- [Custom Datafetcher Context](references/advanced/custom-datafetcher-context.md) — `DgsCustomContextBuilder`, per-request typed context
- [Custom Directives](references/advanced/custom-directives.md) — `@DgsDirective`, `SchemaDirectiveWiring`
- [Custom Object Mapper](references/advanced/custom-object-mapper.md) — Customizing Jackson for DGS deserialization
- [Dynamic Schemas](references/advanced/dynamic-schemas.md) — `@DgsTypeDefinitionRegistry`, `@DgsCodeRegistry`, `ReloadSchemaIndicator`
- [Federated Testing](references/advanced/federated-testing.md) — Testing `_entities` queries
- [File Uploads](references/advanced/file-uploads.md) — `scalar Upload`, `MultipartFile`, multipart dependency
- [GraphQL Context](references/advanced/graphql-context.md) — `GraphQLContextContributor`, context in data loaders
- [Instrumentation](references/advanced/instrumentation.md) — `SimplePerformantInstrumentation`, Apollo Tracing
- [Intercepting HTTP](references/advanced/intercepting-http.md) — `WebGraphQlInterceptor`, headers, response extensions
- [Java Client](references/advanced/java-client.md) — `MonoGraphQLClient`, `GraphQLResponse`, type-safe query API
- [Logging](references/advanced/logging.md) — `notprivacysafe` logger, production settings
- [Operation Caching](references/advanced/operation-caching.md) — `PreparsedDocumentProvider`, Caffeine cache
- [Relay Pagination](references/advanced/relay-pagination.md) — `@connection` directive, `SimpleListConnection`, `PageInfo`
- [Schema Reloading](references/advanced/schema-reloading.md) — Dev mode hot reload, `ReloadSchemaIndicator`
- [Security](references/advanced/security.md) — `@Secured`, `@PreAuthorize`, Spring Security integration
- [Type Resolvers](references/advanced/type-resolvers.md) — `@DgsTypeResolver` for interfaces and unions
- [Virtual Threads](references/advanced/virtual-threads.md) — JDK 21+, `dgsAsyncTaskExecutor`, test considerations

## Key Rules

### Schema design

- ALWAYS design schema first, then implement data fetchers
- ALWAYS place schema files in `src/main/resources/schema/` with `.graphqls` extension
- ALWAYS use input types for mutation arguments: `mutation(input: MyInput!)`
- NEVER expose internal domain objects directly — create dedicated GraphQL types

### Data fetchers

- ALWAYS annotate the data fetcher class with `@DgsComponent`
- USE `@DgsQuery` / `@DgsMutation` as shorthands; use `@DgsData(parentType, field)` for child fetchers
- USE `@InputArgument` on method parameters to bind GraphQL arguments
- PREFER child fetchers with `DgsDataFetchingEnvironment.getSource()` over embedding nested data in parent fetchers
- ALWAYS use `@RequiredArgsConstructor` with Lombok for constructor injection

### Testing

- ALWAYS test data fetchers with `@EnableDgsTest` — it is lighter than `@SpringBootTest` with a full context
- USE `DgsQueryExecutor.executeAndExtractJsonPath()` to extract specific fields
- USE `@MockBean` to mock service dependencies in data fetcher tests
- NEVER rely on the HTTP layer in data fetcher unit tests — use `DgsQueryExecutor` instead

## Ground Rules

- ALWAYS follow schema-first development: schema drives the Java implementation, not the other way around
- ALWAYS keep `@DgsComponent` classes free of business logic — delegate to `@Service` classes
- NEVER add DGS annotations to `@Service` or `@Repository` classes
- PREFER the DGS programming model over Spring GraphQL annotations for consistency
- USE the platform BOM to manage DGS dependency versions — never pin individual module versions manually
