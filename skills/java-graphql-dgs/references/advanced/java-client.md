# DGS Java Client Reference

The DGS framework provides a `GraphQLClient` for calling GraphQL endpoints from Java/Kotlin code, with built-in support for parsing responses, error handling, and JsonPath extraction.

## WebClient-Based Client (Recommended)

```java
WebClient webClient = WebClient.create("http://localhost:8080/graphql");
WebClientGraphQLClient client = MonoGraphQLClient.createWithWebClient(webClient);

Mono<GraphQLResponse> responseMono = client.reactiveExecuteQuery(
    "{ shows { title releaseYear } }"
);

Mono<List<String>> titles = responseMono.map(r ->
    r.extractValue("shows[*].title")
);

// Don't forget to subscribe!
titles.subscribe();
```

Add custom headers:

```java
WebClientGraphQLClient client = MonoGraphQLClient.createWithWebClient(
    webClient,
    headers -> headers.add("Authorization", "Bearer " + token)
);
```

## Client Interfaces

| Interface | Use case |
|---|---|
| `GraphQLClient` | Blocking HTTP clients |
| `MonoGraphQLClient` | Non-blocking, returns `Mono<GraphQLResponse>` |
| `ReactiveGraphQLClient` | Streaming (subscriptions), returns `Flux` |

Built-in implementations: `WebClientGraphQLClient` (via `MonoGraphQLClient.createWithWebClient`).

## Custom HTTP Client

```java
CustomGraphQLClient client = GraphQLClient.createCustom(url,
    (url, headers, body) -> {
        HttpHeaders httpHeaders = new HttpHeaders();
        headers.forEach(httpHeaders::addAll);
        ResponseEntity<String> exchange = restTemplate.exchange(
            url, HttpMethod.POST, new HttpEntity<>(body, httpHeaders), String.class
        );
        return new HttpResponse(exchange.getStatusCodeValue(), exchange.getBody());
    }
);
GraphQLResponse response = client.executeQuery(query, emptyMap(), "MyOperation");
```

## `GraphQLResponse` Methods

| Method | Description |
|---|---|
| `getData()` | Returns data as `Map<String, Object>` |
| `dataAsObject(Class<T>)` | Deserializes data as a Java object |
| `extractValue("path")` | JsonPath extraction (returns type depends on JSON shape) |
| `extractValueAsObject("path", Class<T>)` | JsonPath extraction deserialized to class |
| `extractValueAsObject("path", TypeRef<T>)` | JsonPath extraction for generics (List, Map) |
| `getErrors()` | Returns list of `GraphQLError` |
| `hasErrors()` | True if response contains errors |
| `getParsed()` | Returns `DocumentContext` for further JsonPath processing |

## Type-Safe Query API (with Codegen)

Generate type-safe query builders using the codegen plugin with `generateClient = true`:

```java
ShowsGraphQLQuery query = ShowsGraphQLQuery.newRequest()
    .titleFilter("Ozark")
    .build();

ShowsProjectionRoot<?, ?> projection = new ShowsProjectionRoot<>()
    .title()
    .releaseYear();

String queryString = new GraphQLQueryRequest(query, projection).serialize();
Mono<GraphQLResponse> result = client.reactiveExecuteQuery(queryString);
```

## Subscription Client (SSE / WebSocket)

```java
// SSE subscriptions
SSESubscriptionGraphQLClient sseClient = new SSESubscriptionGraphQLClient("/subscriptions", webClient);

// WebSocket subscriptions
WebSocketGraphQLClient wsClient = new WebSocketGraphQLClient("/subscriptions", webSocketClient);

Flux<GraphQLResponse> events = wsClient.reactiveExecuteQuery(
    "subscription { stocks { name price } }", emptyMap()
);
```
