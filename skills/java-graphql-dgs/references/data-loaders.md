# DGS Data Loaders Reference

Data loaders solve the **N+1 problem**: instead of calling a backend once per parent object, they batch all calls into a single request.

## Implementing a BatchLoader

```java
@DgsDataLoader(name = "directors")
public class DirectorsDataLoader implements BatchLoader<String, Director> {

    @Autowired
    private DirectorServiceClient directorServiceClient;

    @Override
    public CompletionStage<List<Director>> load(List<String> keys) {
        return CompletableFuture.supplyAsync(() -> directorServiceClient.loadDirectors(keys));
    }
}
```

`BatchLoader<K, V>` takes a list of keys and returns a list of values in the **same order**. Use `MappedBatchLoader<K, V>` when not all keys have a value (returns a `Map`).

## MappedBatchLoader

Better when results can be sparse:

```java
@DgsDataLoader(name = "directors")
public class DirectorsDataLoader implements MappedBatchLoader<String, Director> {

    @Override
    public CompletionStage<Map<String, Director>> load(Set<String> keys) {
        return CompletableFuture.supplyAsync(() -> directorServiceClient.loadDirectors(keys));
    }
}
```

## Lambda Style

```java
@DgsComponent
public class DataLoaders {

    @DgsDataLoader(name = "directors")
    public BatchLoader<String, Director> directorLoader =
        keys -> CompletableFuture.supplyAsync(() -> directorServiceClient.loadDirectors(keys));
}
```

## Using a Data Loader in a Child Fetcher

```java
@DgsData(parentType = "Movie", field = "director")
public CompletableFuture<Director> director(DgsDataFetchingEnvironment dfe) {
    // Type-safe lookup using class reference
    DataLoader<String, Director> loader = dfe.getDataLoader(DirectorsDataLoader.class);
    String id = dfe.getSource().getDirectorId();
    return loader.load(id);
}
```

The child fetcher **must return `CompletableFuture`** — this is how the framework batches requests across all parent objects before calling the loader.

## Handling Partial Failures

Return `Try<V>` to allow partial success — failed items produce GraphQL errors while successful items return data:

```java
@DgsDataLoader(name = "directors")
public class DirectorsDataLoader implements BatchLoader<String, Try<Director>> {

    @Override
    public CompletionStage<List<Try<Director>>> load(List<String> keys) {
        return CompletableFuture.supplyAsync(() ->
            keys.stream()
                .map(key -> Try.tryCall(() -> directorServiceClient.load(key)))
                .collect(Collectors.toList())
        );
    }
}
```

## Accessing Custom Context in a DataLoader

```java
@DgsDataLoader(name = "exampleLoaderWithContext")
public class ExampleLoader implements BatchLoaderWithContext<String, String> {

    @Override
    public CompletionStage<List<String>> load(List<String> keys, BatchLoaderEnvironment env) {
        MyContext context = DgsContext.getCustomContext(env);
        return CompletableFuture.supplyAsync(() ->
            keys.stream().map(k -> context.getCustomState() + " " + k).collect(Collectors.toList())
        );
    }
}
```

## Virtual Threads with Data Loaders

When `dgs.graphql.virtualthreads.enabled=true` is set, data loaders do NOT automatically run on virtual threads (due to the `CompletableFuture` API). Pass the DGS executor explicitly:

```java
@Autowired
@Qualifier("dgsAsyncTaskExecutor")
Executor executor;

// In the data loader:
return CompletableFuture.supplyAsync(() -> loadData(keys), executor);
```
