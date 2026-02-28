# DGS GraphQL Context Leveraging Reference

## GraphQL Context vs Custom Context

The **GraphQL context** (`GraphQLContext`) is built-in to graphql-java and shared across all data fetchers in a request. The **custom context** (`DgsCustomContextBuilder`) is DGS-specific and strongly typed.

Use the GraphQL context when you need to contribute values from HTTP request interceptors.

## `GraphQLContextContributor`

Implement `GraphQLContextContributor` to add values to the `GraphQLContext` at the start of each request:

```java
@Component
public class RequestHeaderContextContributor implements GraphQLContextContributor {

    @Override
    public void contribute(
            GraphQLContext.Builder builder,
            @Nullable Map<String, Object> extensions,
            @Nullable DgsRequestData dgsRequestData) {

        if (dgsRequestData != null && dgsRequestData.getHeaders() != null) {
            String traceId = dgsRequestData.getHeaders().getFirst("X-Trace-Id");
            builder.put("traceId", traceId);
        }
    }
}
```

## Accessing GraphQL Context in Data Fetchers

```java
@DgsQuery
public Show show(@InputArgument String id, DataFetchingEnvironment dfe) {
    GraphQLContext ctx = dfe.getGraphQlContext();
    String traceId = ctx.get("traceId");
    // Use trace ID for logging
    return showService.findById(id);
}
```

## Accessing GraphQL Context in Data Loaders

Data loaders receive the context through `BatchLoaderEnvironment`:

```java
@DgsDataLoader(name = "shows")
public class ShowsDataLoader implements BatchLoaderWithContext<String, Show> {

    @Override
    public CompletionStage<List<Show>> load(List<String> keys, BatchLoaderEnvironment env) {
        GraphQLContext gqlContext = env.getContext();
        String traceId = gqlContext.get("traceId");
        return CompletableFuture.supplyAsync(() -> showService.load(keys, traceId));
    }
}
```

## Difference: GraphQLContext vs LocalContext

| | `GraphQLContext` | `DataFetcherResult.localContext` |
|---|---|---|
| Scope | Whole request | Sub-tree only |
| Set by | `GraphQLContextContributor` | Parent data fetcher |
| Accessed via | `dfe.getGraphQlContext()` | `dfe.getLocalContext()` |
| Use case | Request headers, auth, tracing | Passing computed data to child fetchers |
