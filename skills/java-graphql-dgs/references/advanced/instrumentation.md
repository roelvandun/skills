# DGS Instrumentation Reference

## Overview

DGS supports graphql-java `Instrumentation` for tracing, metrics, and monitoring query execution.

## Creating a Custom Instrumentation

Extend `SimplePerformantInstrumentation` (preferred over `SimpleInstrumentation` for performance):

```java
@Component
public class RequestLoggingInstrumentation extends SimplePerformantInstrumentation {

    private static final Logger log = LoggerFactory.getLogger(RequestLoggingInstrumentation.class);

    @Override
    public InstrumentationContext<ExecutionResult> beginExecution(
            InstrumentationExecutionParameters parameters,
            InstrumentationState state) {

        long start = System.currentTimeMillis();
        return SimpleInstrumentationContext.whenCompleted((result, ex) -> {
            long elapsed = System.currentTimeMillis() - start;
            log.info("Query completed in {}ms: {}", elapsed, parameters.getQuery());
        });
    }
}
```

Register as `@Component` — DGS auto-discovers it.

## Key Instrumentation Hooks

| Method | Fires when |
|---|---|
| `beginExecution` | Before the full query execution |
| `beginParse` | Before the query is parsed |
| `beginValidation` | Before the query is validated |
| `beginExecuteOperation` | Before field resolution starts |
| `beginField` | Before each field is resolved |
| `beginFieldFetch` | Before each data fetcher is called |

## Handling Async Results

For async data fetchers (returning `CompletableFuture`), use the correct instrumentation context:

```java
@Override
public InstrumentationContext<Object> beginFieldFetch(
        InstrumentationFieldFetchParameters parameters,
        InstrumentationState state) {

    return new SimpleInstrumentationContext<>() {
        @Override
        public void onCompleted(Object result, Throwable t) {
            // This fires after the CompletableFuture completes
            recordMetric(parameters.getField().getName(), t == null);
        }
    };
}
```

## Apollo Tracing Instrumentation

To enable Apollo Tracing (sends timing data in the response `extensions`):

```java
@Bean
public TracingInstrumentation tracingInstrumentation() {
    return new TracingInstrumentation();
}
```

This adds an `extensions.tracing` block to every response with per-field timing.

## Chaining Multiple Instrumentations

DGS automatically chains multiple `Instrumentation` beans using `ChainedInstrumentation`:

```java
@Component
public class MetricsInstrumentation extends SimplePerformantInstrumentation { ... }

@Component
public class TracingInstrumentation extends SimplePerformantInstrumentation { ... }
```

Both are active simultaneously — no additional configuration needed.

## DGS Built-in Metrics Instrumentation

DGS includes metrics instrumentation by default (requires Micrometer). Disable with:

```yaml
management:
  metrics:
    dgs-graphql:
      instrumentation:
        enabled: false
```
