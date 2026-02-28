# DGS Logging Reference

## Controlling graphql-java Query Logging

The `notprivacysafe` SLF4J logger (from graphql-java) logs errors, invalid queries, and optionally full query text.

**Disable all graphql-java logging** (recommended in production to prevent accidental PII logging):

```yaml
logging:
  level:
    notprivacysafe: OFF
```

**Enable debug logging** (logs query text at parse/validation/execution stages):

```yaml
logging:
  level:
    notprivacysafe: DEBUG
```

## Why "notprivacysafe"?

The logger name `notprivacysafe` is intentional — it signals that the data logged at `DEBUG` level may contain query variables and user-submitted input, which may include personal information. By forcing this name, graphql-java makes it easy to configure this logger separately.

## DGS Framework Logging

DGS uses standard SLF4J loggers under the `com.netflix.graphql.dgs` package:

```yaml
logging:
  level:
    com.netflix.graphql.dgs: DEBUG
```

## Recommended Production Settings

```yaml
logging:
  level:
    # Suppress potentially sensitive query/variable logging
    notprivacysafe: OFF
    # DGS framework logs (warnings/errors only in production)
    com.netflix.graphql.dgs: WARN
```

## Logging in Custom Instrumentation

For request-level logging with timing, use a custom `Instrumentation` (see `advanced/instrumentation.md`):

```java
@Component
public class QueryLoggingInstrumentation extends SimplePerformantInstrumentation {

    @Override
    public InstrumentationContext<ExecutionResult> beginExecution(
            InstrumentationExecutionParameters params, InstrumentationState state) {
        long start = System.currentTimeMillis();
        String operationName = params.getOperation();
        return SimpleInstrumentationContext.whenCompleted((result, ex) -> {
            log.info("GraphQL {} completed in {}ms with {} errors",
                operationName, System.currentTimeMillis() - start, result.getErrors().size());
        });
    }
}
```
