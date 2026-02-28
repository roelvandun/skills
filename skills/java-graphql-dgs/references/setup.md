# DGS Setup Reference

## Requirements

- Spring Boot 3.x
- Java 17+
- Maven 3.8+

## Maven Dependencies

### Parent / root `pom.xml` — add to `<dependencyManagement>`

```xml
<dependencyManagement>
  <dependencies>
    <!-- DGS platform BOM — aligns all DGS module versions -->
    <dependency>
      <groupId>com.netflix.graphql.dgs</groupId>
      <artifactId>graphql-dgs-platform-dependencies</artifactId>
      <version>10.0.6</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
```

### Module `pom.xml` — add to `<dependencies>`

```xml
<!-- DGS starter (includes Spring for GraphQL) -->
<dependency>
  <groupId>com.netflix.graphql.dgs</groupId>
  <artifactId>dgs-starter</artifactId>
</dependency>

<!-- For tests -->
<dependency>
  <groupId>com.netflix.graphql.dgs</groupId>
  <artifactId>dgs-starter-test</artifactId>
  <scope>test</scope>
</dependency>
```

The `dgs-starter` already pulls in `spring-boot-starter-web`, so you do not need to add it separately if you only need the GraphQL endpoint.

## Schema Files

Place all `.graphqls` schema files in:

```
src/main/resources/schema/
```

DGS scans this directory automatically. You can split the schema across multiple files — DGS merges them.

Example directory layout:

```
src/main/resources/
└── schema/
    ├── user.graphqls
    ├── loyalty.graphqls
    └── scalars.graphqls
```

## Endpoints

| Endpoint | Purpose |
|---|---|
| `POST /graphql` | GraphQL query endpoint |
| `GET /graphiql` | Interactive query editor (dev only) |

To disable GraphiQL in production, add to `application.yml`:

```yaml
spring:
  graphql:
    graphiql:
      enabled: false
```

## Custom Scalars

Register custom scalars as Spring beans:

```java
@Bean
public RuntimeWiringConfigurer customScalarConfigurer() {
    return builder -> builder.scalar(ExtendedScalars.Date);
}
```

Add the `graphql-java-extended-scalars` dependency for common scalars (Date, DateTime, JSON, etc.).

## Multi-module Projects

When adding DGS to a module in an existing multi-module Maven project:

1. Add the BOM import to the root `pom.xml` `<dependencyManagement>`.
2. Add `dgs-starter` (no version needed) to the module's `pom.xml`.
3. Create the `src/main/resources/schema/` directory in the module.
4. The main `@SpringBootApplication` class remains unchanged.
