# DGS Virtual Threads Reference

Virtual Threads (JDK 21+) enable concurrent data fetcher execution without managing thread pools.

## Default Threading Model (Without Virtual Threads)

Without virtual threads, all data fetchers run on the **same request thread** sequentially. Concurrency requires explicitly returning `CompletableFuture` and managing your own `Executor`.

## Enabling Virtual Threads for DGS

Requires JDK 21+. Each user-defined data fetcher runs in its own virtual thread, providing automatic concurrency:

```yaml
dgs:
  graphql:
    virtualthreads:
      enabled: true
```

"User-defined" means data fetchers annotated with `@DgsQuery`, `@DgsMutation`, `@DgsData`, etc. Trivial POJO field fetchers do NOT run in separate virtual threads.

## Enabling Virtual Threads for Spring WebMVC

Separate from DGS virtual threads — controls Tomcat request handling threads:

```yaml
spring:
  threads:
    virtual:
      enabled: true
```

Combining both settings gives a fully virtual-thread-based request stack (requires Spring Boot 3.2+).

## Data Loaders with Virtual Threads

Data loaders use `CompletableFuture` and cannot automatically use virtual threads. Inject the DGS virtual thread executor:

```java
@Autowired
@Qualifier("dgsAsyncTaskExecutor")
Executor executor;

// In a BatchLoader or MappedBatchLoader:
@Override
public CompletionStage<List<Show>> load(List<String> keys) {
    return CompletableFuture.supplyAsync(() -> showService.load(keys), executor);
}
```

Data fetchers that **explicitly** return `CompletableFuture` are also excluded from automatic virtual thread wrapping — they still use the executor you provide.

## ThreadLocal and Context Propagation

Virtual threads work like platform threads for `ThreadLocal` access, but data fetchers run on **different threads** from the caller. This affects:

- **Spring Security context**: Configure `InheritableThreadLocal` strategy
- **MDC logging context**: Use context propagation or pass data explicitly
- **`@Transactional` in tests**: Transaction rollback does not span threads — do not rely on auto-rollback in `@SpringBootTest` with virtual threads enabled

## Test Considerations

When using `DgsQueryExecutor` in tests with virtual threads enabled, data fetchers run in different threads. If your test relies on `@Transactional` rollback:

1. **Preferred**: Don't rely on transaction rollback; use `@Sql` for test data setup/teardown
2. **Alternative**: Disable virtual threads in the test: `@SpringBootTest(properties = "dgs.graphql.virtualthreads.enabled=false")`
