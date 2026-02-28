# DGS Error Handling Reference

## Built-in Exception Mapping

The DGS framework maps exceptions from data fetchers to GraphQL errors automatically:

| Exception | GraphQL error type |
|---|---|
| Any `RuntimeException` | `INTERNAL` |
| `AccessDeniedException` | `PERMISSION_DENIED` |
| `DgsEntityNotFoundException` | `NOT_FOUND` |

## Option 1: `@ControllerAdvice` (Recommended)

The simplest approach for custom exception mapping:

```java
@ControllerAdvice
public class ControllerExceptionHandler {

    @GraphQlExceptionHandler
    public GraphQLError handle(IllegalArgumentException ex) {
        return GraphQLError.newError()
            .errorType(ErrorType.BAD_REQUEST)
            .message(ex.getMessage())
            .build();
    }
}
```

If no matching handler is found, the built-in handler takes over.

## Option 2: `DataFetcherExceptionHandler`

More control, but requires delegating to `DefaultDataFetcherExceptionHandler` for unhandled cases:

```java
@Component
public class MyExceptionHandler implements DataFetcherExceptionHandler {

    private final DefaultDataFetcherExceptionHandler defaultHandler = new DefaultDataFetcherExceptionHandler();

    @Override
    public CompletableFuture<DataFetcherExceptionHandlerResult> handleException(
            DataFetcherExceptionHandlerParameters params) {

        if (params.getException() instanceof CustomerNotFoundException) {
            GraphQLError error = TypedGraphQLError.newNotFoundBuilder()
                .message(params.getException().getMessage())
                .path(params.getPath())
                .build();
            return CompletableFuture.completedFuture(
                DataFetcherExceptionHandlerResult.newResult().error(error).build()
            );
        }
        return defaultHandler.handleException(params);
    }
}
```

## Subscription Exceptions

Subscription errors require a `SubscriptionExceptionResolver` (the `DataFetcherExceptionHandler` does not cover `Publisher` emissions):

```java
@Component
public class MySubscriptionExceptionResolver implements SubscriptionExceptionResolver {

    @Override
    public Mono<List<GraphQLError>> resolveException(Throwable exception) {
        if (exception instanceof MyCustomException) {
            return Mono.just(List.of(
                GraphQLError.newError()
                    .message(exception.getMessage())
                    .errorType(ErrorType.INTERNAL)
                    .build()
            ));
        }
        return Mono.just(List.of(GraphqlErrorBuilder.newError().message(exception.getMessage()).build()));
    }
}
```

## Error Response Structure

GraphQL errors appear in the `errors` array alongside partial data:

```json
{
  "errors": [{
    "message": "Customer not found",
    "locations": [],
    "path": ["user"],
    "extensions": {
      "errorType": "NOT_FOUND"
    }
  }],
  "data": { "user": null }
}
```

## Error Categories

- **Comprehensive Errors** — unexpected technical errors; appear in `errors[]`; client cannot fix
- **Errors as Data** — domain errors meaningful to the user (e.g. "account suspended"); modeled as fields in the schema using union result types
