# DGS Context Passing Reference

## Getting the Parent Object (`getSource`)

Inside a child data fetcher, call `dfe.getSource()` to access the parent object:

```java
@DgsData(parentType = "Movie", field = "director")
public Director director(DgsDataFetchingEnvironment dfe) {
    Movie movie = dfe.getSource();  // The parent Movie object
    return directorService.findById(movie.getDirectorId());
}
```

## Passing Local Context Without Schema Fields

Use `DataFetcherResult<T>` with `localContext` to pass data to child fetchers without exposing it in the schema:

```java
@DgsQuery
public DataFetcherResult<Show> show(@InputArgument String id, DataFetchingEnvironment dfe) {
    Show show = showService.findById(id);
    return DataFetcherResult.<Show>newResult()
        .data(show)
        .localContext(Map.of("showId", id, "requestedBy", dfe.getVariables().get("user")))
        .build();
}

@DgsData(parentType = "Show", field = "reviews")
public List<Review> reviews(DataFetchingEnvironment dfe) {
    Map<String, String> ctx = dfe.getLocalContext();
    return reviewService.findByShow(ctx.get("showId"));
}
```

`localContext` is separate from the main GraphQL context — it's scoped to the sub-tree of the query execution, not shared across unrelated data fetchers.

## Pre-loading Data Based on Selection Set

Avoid over-fetching by checking if a field is actually requested:

```java
@DgsQuery
public Show show(@InputArgument String id, DataFetchingEnvironment dfe) {
    Show show = showService.findById(id);
    // Only fetch reviews if the client requested them
    if (dfe.getSelectionSet().contains("reviews")) {
        show.setReviews(reviewService.findByShow(id));
    }
    return show;
}
```

## DataFetcherResult vs DataLoader

| Approach | When to use |
|---|---|
| `DataFetcherResult.localContext` | Passing computed values to child fetchers in the same tree |
| `DataLoader` | Batching many requests to the same backend across all parent objects |
| `getSource()` | Simple child resolution when parent already has the needed key |
