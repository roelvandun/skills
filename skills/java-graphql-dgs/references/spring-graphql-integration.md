# DGS Spring for GraphQL Integration Reference

## DGS Now Uses Spring for GraphQL Internally

As of DGS 7+, the DGS framework delegates to Spring for GraphQL internally. Use DGS annotations (`@DgsComponent`, `@DgsQuery`, etc.) consistently — do **not** mix with `@QueryMapping` or `@SchemaMapping` from Spring for GraphQL.

## Correct Starter

```xml
<dependency>
    <groupId>com.netflix.graphql.dgs</groupId>
    <artifactId>dgs-starter</artifactId>
</dependency>
```

This pulls in Spring for GraphQL automatically. Do NOT add `spring-boot-starter-graphql` separately.

## Overlapping Configuration Properties

Both DGS and Spring for GraphQL expose similar properties. DGS properties take precedence where they overlap:

| Concern | DGS property | Spring GraphQL property |
|---|---|---|
| GraphQL endpoint path | `dgs.graphql.path` | `spring.graphql.path` |
| GraphiQL UI | `dgs.graphql.graphiql.enabled` | `spring.graphql.graphiql.enabled` |
| Introspection | `dgs.graphql.introspection.enabled` | `spring.graphql.schema.introspection.enabled` |

When using DGS, prefer `dgs.graphql.*` properties.

## Schema Source

DGS resolves schemas from `src/main/resources/schema/*.graphqls` by default. Configure with:

```yaml
dgs:
  graphql:
    schema-locations:
      - classpath*:schema/**/*.graphqls
```

## WebSocket Subscriptions

DGS uses Spring for GraphQL's WebSocket transport. The `spring-boot-starter-websocket` dependency is required for WebMVC (not needed for WebFlux):

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
```

Enable the WebSocket endpoint:

```yaml
spring:
  graphql:
    websocket:
      path: /subscriptions
```

## File Uploads

Add the `multipart-spring-graphql` library when handling file uploads with DGS + Spring for GraphQL:

```xml
<dependency>
    <groupId>name.nkonev.multipart-spring-graphql</groupId>
    <artifactId>multipart-spring-graphql</artifactId>
</dependency>
```

Declare `scalar Upload` in your schema, and use `MultipartFile` in your data fetcher:

```java
@DgsMutation
public String upload(@InputArgument("file") MultipartFile file) throws IOException {
    // process file
}
```

## Testing

Testing does not require Spring for GraphQL's `GraphQlTester`. DGS's `DgsQueryExecutor` and `@EnableDgsTest` continue to work as before. Use `WebSocketGraphQlTester` only for WebSocket integration tests.
