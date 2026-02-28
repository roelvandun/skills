# DGS Code Generation Reference

Code generation creates POJOs, enums, input types, and type-safe query builders from your GraphQL schema.

## Gradle Setup

```groovy
plugins {
    id "com.netflix.dgs.codegen" version "7.0.3"
}

generateJava {
    schemaPaths = ["${projectDir}/src/main/resources/schema"]
    packageName = "com.example.demo.generated"
    generateClient = true       // Generate type-safe query builders
    generateInterfaces = false  // Generate interfaces for types
    typeMapping = [
        "DateTime": "java.time.LocalDateTime",
        "PositiveInt": "java.lang.Integer"
    ]
}
```

Run `./gradlew generateJava` to generate code. The task runs automatically before `compileJava`.

## Maven Setup

Community plugin (not officially maintained by Netflix):

```xml
<plugin>
    <groupId>io.github.deweyjose</groupId>
    <artifactId>graphqlcodegen-maven-plugin</artifactId>
    <version>1.50</version>
    <executions>
        <execution>
            <goals><goal>generate</goal></goals>
        </execution>
    </executions>
    <configuration>
        <schemaPaths>src/main/resources/schema</schemaPaths>
        <packageName>com.example.demo.generated</packageName>
    </configuration>
</plugin>
```

## What Gets Generated

| Schema element | Generated output |
|---|---|
| `type Foo` | `Foo.java` POJO with getters/setters/builder |
| `input FooInput` | `FooInput.java` |
| `enum FooEnum` | `FooEnum.java` |
| `interface IFoo` | `IFoo.java` interface (if `generateInterfaces=true`) |
| All types | `DgsConstants.java` with string constants for field names |

## DgsConstants

Use `DgsConstants` for safe string references:

```java
@DgsData(parentType = DgsConstants.QUERY_TYPE, field = DgsConstants.QUERY.Shows)
public List<Show> shows() { ... }
```

## Type-Safe Query Builders

With `generateClient = true`, generate type-safe query request/response objects:

```java
ShowsGraphQLQuery query = ShowsGraphQLQuery.newRequest().build();
ShowsProjectionRoot projection = new ShowsProjectionRoot<>()
    .title()
    .releaseYear();

GraphQLQueryRequest request = new GraphQLQueryRequest(query, projection);
String queryString = request.serialize();
```

Use with `DgsQueryExecutor` in tests:

```java
GraphQLQueryRequest queryRequest = new GraphQLQueryRequest(
    ShowsGraphQLQuery.newRequest().titleFilter("Ozark").build(),
    new ShowsProjectionRoot<>().title()
);
List<String> titles = queryExecutor.executeAndExtractJsonPath(
    queryRequest.serialize(), "$.data.shows[*].title"
);
```

## Key Configuration Options

| Option | Description |
|---|---|
| `generateClient` | Generate type-safe query builders |
| `generateInterfaces` | Generate interfaces for types (useful for DGS federation) |
| `typeMapping` | Map GraphQL scalar names to Java class names |
| `trackInputFieldSet` | Generate `has[Field]()` methods for partial input detection |
| `generateJSpecifyAnnotations` | Add `@Nullable`/`@NonNull` JSpecify annotations |
| `generateCustomAnnotations` | Custom annotations from schema directives |
| `skipEntityQueries` | Skip generation of `_entities` query types |

## Generated Output Location

Generated sources are placed in `build/generated/sources/dgsCodegen/` and auto-added to the compile source set.
