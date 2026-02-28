# DGS Relay Pagination Reference

DGS can dynamically generate Relay-compliant `Connection` and `Edge` types from a `@connection` directive.

## Setup

Add the pagination module:

```xml
<dependency>
    <groupId>com.netflix.graphql.dgs</groupId>
    <artifactId>graphql-dgs-pagination</artifactId>
</dependency>
```

## Schema

Annotate the type you want to paginate with `@connection`. DGS generates `MessageConnection`, `MessageEdge`, and `PageInfo` automatically:

```graphql
type Query {
    messages: MessageConnection
}
type Message @connection {
    text: String
}
```

No need to define `MessageConnection`, `MessageEdge`, or `PageInfo` in your schema — they are dynamically added.

## Data Fetcher

Use the `graphql.relay` types (`Connection<T>`, `Edge<T>`, `PageInfo`):

```java
@DgsData(parentType = "Query", field = "messages")
public Connection<Message> messages(DataFetchingEnvironment env) {
    List<Message> messages = messageService.getAll();
    return new SimpleListConnection<>(messages).get(env);
}
```

`SimpleListConnection` takes a `List<T>` and handles cursor calculation and `PageInfo` automatically.

## Custom Cursor Pagination

For keyset/cursor-based pagination with a real database:

```java
@DgsData(parentType = "Query", field = "messages")
public Connection<Message> messages(@InputArgument String after, @InputArgument Integer first) {
    List<Message> page = messageService.findAfterCursor(after, first + 1);
    boolean hasNextPage = page.size() > first;
    List<Message> nodes = hasNextPage ? page.subList(0, first) : page;

    List<Edge<Message>> edges = nodes.stream()
        .map(m -> new DefaultEdge<>(m, new DefaultConnectionCursor(m.getId())))
        .collect(Collectors.toList());

    PageInfo pageInfo = new DefaultPageInfo(
        edges.isEmpty() ? null : edges.get(0).getCursor(),
        edges.isEmpty() ? null : edges.get(edges.size() - 1).getCursor(),
        after != null,
        hasNextPage
    );
    return new DefaultConnection<>(edges, pageInfo);
}
```

## Testing

Include `DgsPaginationTypeDefinitionRegistry` and `PageInfo` in the test context:

```java
@SpringBootTest(classes = {
    DgsAutoConfiguration.class,
    DgsPaginationTypeDefinitionRegistry.class,
    PageInfo.class,
    MessageDataFetcher.class
})
@EnableDgsTest
class MessagePaginationTest { ... }
```

## Codegen Type Mapping

If using the codegen plugin, map the generated connection type:

```groovy
generateJava {
    typeMapping = [
        "MessageConnection": "graphql.relay.SimpleListConnection<Message>"
    ]
}
```

## Note on Federation

The `@connection` directive only works when your service does **not** need to register a static schema file with an external gateway. Some federation gateways may not recognize dynamically generated types.
