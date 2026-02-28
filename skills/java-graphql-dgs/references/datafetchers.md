# DGS Data Fetchers Reference

## Core Annotations

| Annotation | Equivalent to | When to use |
|---|---|---|
| `@DgsComponent` | `@Component` | Mark a class as containing data fetchers |
| `@DgsQuery` | `@DgsData(parentType="Query")` | Query field on the root `Query` type |
| `@DgsMutation` | `@DgsData(parentType="Mutation")` | Mutation field on the root `Mutation` type |
| `@DgsSubscription` | `@DgsData(parentType="Subscription")` | Subscription field |
| `@DgsData` | — | Any field on any type; required for child fetchers |
| `@InputArgument` | — | Bind a method parameter to a GraphQL input argument |

## Basic Query Fetcher

```java
@DgsComponent
@RequiredArgsConstructor
public class ShowsDataFetcher {

    private final ShowService showService;

    @DgsQuery
    public List<Show> shows(@InputArgument String titleFilter) {
        return showService.findAll(titleFilter);
    }
}
```

The method name becomes the field name by default. Override with `@DgsQuery(field = "myField")`.

## Input Types

For complex input arguments, define an `input` type in the schema and bind it directly:

```graphql
input CreateShowInput {
  title: String!
  releaseYear: Int!
}
```

```java
@DgsMutation
public Show createShow(@InputArgument CreateShowInput input) {
    return showService.create(input.getTitle(), input.getReleaseYear());
}
```

DGS automatically deserializes the GraphQL input object to the Java class. The Java class must have matching field names (camelCase).

## Child Data Fetchers

Use `@DgsData` with `parentType` to fetch fields that are expensive to load:

```graphql
type Show {
  id: ID!
  title: String!
  actors: [Actor!]!    # loaded by a child fetcher
}
```

```java
@DgsData(parentType = "Show", field = "actors")
public List<Actor> actors(DgsDataFetchingEnvironment dfe) {
    Show show = dfe.getSource();      // get the parent Show object
    return actorService.findByShowId(show.getId());
}
```

The framework calls the child fetcher once per parent object, which causes N+1 queries. Use data loaders to batch these calls.

## Data Loaders

Data loaders batch multiple calls into one to solve the N+1 problem:

```java
@DgsDataLoader(name = "actors")
public class ActorsDataLoader implements BatchLoader<String, List<Actor>> {

    private final ActorService actorService;

    @Override
    public CompletionStage<List<List<Actor>>> load(List<String> showIds) {
        return CompletableFuture.supplyAsync(() ->
            actorService.findByShowIds(showIds)
        );
    }
}
```

Use the loader in the child fetcher:

```java
@DgsData(parentType = "Show", field = "actors")
public CompletableFuture<List<Actor>> actors(DgsDataFetchingEnvironment dfe) {
    DataLoader<String, List<Actor>> loader = dfe.getDataLoader(ActorsDataLoader.class);
    Show show = dfe.getSource();
    return loader.load(show.getId());
}
```

## Multiple Fields on One Method

Use `@DgsData.List` to map multiple schema fields to the same implementation:

```java
@DgsData.List({
    @DgsData(parentType = "Query", field = "movies"),
    @DgsData(parentType = "Query", field = "shows")
})
public List<Content> content() {
    return contentService.findAll();
}
```

Note: `@DgsQuery` and `@DgsMutation` cannot be stacked — use `@DgsData` explicitly.

## Accessing Context

The `DgsDataFetchingEnvironment` provides access to the full request context:

```java
@DgsQuery
public User me(DgsDataFetchingEnvironment dfe) {
    // HTTP headers
    String token = dfe.getGraphQlContext().get("Authorization");
    return userService.getCurrentUser(token);
}
```

## Error Handling

Throw typed exceptions from data fetchers. Map them to GraphQL errors in a `DataFetcherExceptionHandler` bean:

```java
@Component
public class CustomExceptionHandler extends DefaultDataFetcherExceptionHandler {

    @Override
    public DataFetcherExceptionHandlerResult onException(DataFetcherExceptionHandlerParameters params) {
        Throwable exception = params.getException();
        if (exception instanceof CustomerNotFoundException) {
            GraphQLError error = TypedGraphQLError.newNotFoundBuilder()
                .message(exception.getMessage())
                .path(params.getPath())
                .build();
            return DataFetcherExceptionHandlerResult.newResult(error).build();
        }
        return super.onException(params);
    }
}
```
