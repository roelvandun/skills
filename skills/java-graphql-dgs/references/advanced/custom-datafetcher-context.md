# DGS Custom Datafetcher Context Reference

## Overview

Custom context allows you to pass request-scoped data (e.g., from HTTP headers) to all data fetchers without using `ThreadLocal` or passing it as parameters.

## Implementing a Custom Context

1. **Create a context class:**

```java
public class MyContext {
    private final String userId;

    public MyContext(String userId) { this.userId = userId; }
    public String getUserId() { return userId; }
}
```

2. **Implement `DgsCustomContextBuilder<T>`:**

```java
@Component
public class MyContextBuilder implements DgsCustomContextBuilder<MyContext> {

    @Override
    public MyContext build() {
        // Called once per request; access HTTP request via RequestContextHolder
        ServletRequestAttributes attrs =
            (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        String userId = attrs.getRequest().getHeader("X-User-Id");
        return new MyContext(userId);
    }
}
```

3. **Retrieve the context in a data fetcher:**

```java
@DgsQuery
public Show show(@InputArgument String id, DgsDataFetchingEnvironment dfe) {
    MyContext ctx = DgsContext.getCustomContext(dfe);
    log.info("Request from user: {}", ctx.getUserId());
    return showService.findById(id);
}
```

## Accessing Custom Context in a BatchLoader

Use `BatchLoaderWithContext` and `DgsContext.getCustomContext(env)`:

```java
@DgsDataLoader(name = "shows")
public class ShowsDataLoader implements BatchLoaderWithContext<String, Show> {

    @Override
    public CompletionStage<List<Show>> load(List<String> keys, BatchLoaderEnvironment env) {
        MyContext ctx = DgsContext.getCustomContext(env);
        return CompletableFuture.supplyAsync(() ->
            showService.loadShows(keys, ctx.getUserId())
        );
    }
}
```

## Notes

- `DgsCustomContextBuilder.build()` is called **once per GraphQL request**
- The context object is immutable by convention — build it from request headers/session in `build()`
- Works for both blocking and reactive implementations
- Use this instead of `RequestContextHolder` inside data loaders (which may run on different threads)
