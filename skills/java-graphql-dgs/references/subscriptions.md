# DGS Subscriptions Reference

Subscriptions push continuous data from server to client using reactive streams.

## Implementing a Subscription

Use `@DgsSubscription` (shorthand for `@DgsData(parentType = "Subscription")`). Must return a `Publisher<T>`:

```java
import reactor.core.publisher.Flux;
import org.reactivestreams.Publisher;

@DgsComponent
public class StockSubscription {

    @DgsSubscription
    public Publisher<Stock> stocks() {
        return Flux.interval(Duration.ofSeconds(1))
            .map(t -> new Stock("NFLX", 500 + t));
    }
}
```

Schema:

```graphql
type Subscription {
    stocks: Stock
}
```

## Transport: WebSockets

Uses the `graphql-transport-ws` sub-protocol via Spring for GraphQL.

**Enable WebSocket path** (required):

```yaml
spring:
  graphql:
    websocket:
      path: /subscriptions
```

**Add WebMVC dependency** (not needed for Webflux):

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
```

## Transport: Server-Sent Events (SSE)

No extra config or dependencies needed. Use the regular `/graphql` endpoint with `text/event-stream` media type.

## Unit Testing Subscriptions

Use `DgsQueryExecutor` — a subscription returns a `Publisher<ExecutionResult>`:

```java
@SpringBootTest(classes = {StockSubscription.class, DgsAutoConfiguration.class})
@EnableDgsTest
class StockSubscriptionTest {

    @Autowired DgsQueryExecutor queryExecutor;
    ObjectMapper objectMapper = new ObjectMapper();

    @Test
    void stocks() {
        ExecutionResult result = queryExecutor.execute("subscription { stocks { name price } }");
        Publisher<ExecutionResult> publisher = result.getData();

        VirtualTimeScheduler vts = VirtualTimeScheduler.create();
        StepVerifier.withVirtualTime(() -> publisher, 3)
            .expectSubscription()
            .thenRequest(3)
            .assertNext(r -> assertThat(toStock(r).getPrice()).isEqualTo(500))
            .assertNext(r -> assertThat(toStock(r).getPrice()).isEqualTo(501))
            .thenCancel()
            .verify();
    }

    private Stock toStock(ExecutionResult result) {
        Map<String, Object> data = result.getData();
        return objectMapper.convertValue(data.get("stocks"), Stock.class);
    }
}
```

Each `onNext` from the `Publisher` is a nested `ExecutionResult` with the actual data.

## Integration Testing Subscriptions

Use `WebSocketGraphQlTester` with a full `@SpringBootTest`:

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class SubscriptionIntegrationTest {

    @LocalServerPort int port;
    GraphQlTester graphQlTester;

    @BeforeEach
    void setUp() {
        URI url = URI.create("http://localhost:" + port + "/subscriptions");
        this.graphQlTester = WebSocketGraphQlTester
            .builder(url, new ReactorNettyWebSocketClient()).build();
    }

    @Test
    void stocks() {
        Flux<Stock> result = graphQlTester
            .document("subscription { stocks { name price } }")
            .executeSubscription()
            .toFlux("stocks", Stock.class);

        StepVerifier.create(result)
            .assertNext(s -> assertThat(s.getPrice()).isEqualTo(500))
            .thenCancel().verify();
    }
}
```

## Exception Handling in Subscriptions

Use `SubscriptionExceptionResolver` — the regular `DataFetcherExceptionHandler` does **not** apply to `Publisher` emissions. See `error-handling.md`.
