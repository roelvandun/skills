# DGS Configuration Reference

All DGS properties are under `dgs.graphql.*` in `application.yml`.

## Core Properties

| Property | Default | Description |
|---|---|---|
| `dgs.graphql.path` | `/graphql` | GraphQL HTTP endpoint path |
| `dgs.graphql.schema-locations` | `classpath*:schema/**/*.graphqls` | Schema file locations |
| `dgs.graphql.introspection.enabled` | `true` | Enable/disable schema introspection |
| `dgs.graphql.graphiql.enabled` | `true` | Enable the GraphiQL UI |
| `dgs.graphql.graphiql.path` | `/graphiql` | Path for GraphiQL UI |

## WebSocket / Subscriptions

| Property | Default | Description |
|---|---|---|
| `spring.graphql.websocket.path` | _(none)_ | Enable WebSocket subscriptions at this path |
| `spring.graphql.websocket.connection-init-timeout` | `60s` | Timeout waiting for `connection_init` |

## Virtual Threads (JDK 21+)

| Property | Default | Description |
|---|---|---|
| `dgs.graphql.virtualthreads.enabled` | `false` | Run each user data fetcher on a virtual thread |
| `spring.threads.virtual.enabled` | `false` | Use virtual threads for Spring WebMVC request threads |

## Development / Hot Reload

| Property | Default | Description |
|---|---|---|
| `dgs.reload` | `false` | Re-initialize schema on every request (dev mode) |

Alternatively, enable the `laptop` Spring profile to activate dev mode automatically.

## Metrics

| Property | Default | Description |
|---|---|---|
| `management.metrics.dgs-graphql.enabled` | `true` | Enable DGS-specific metrics |
| `management.metrics.dgs-graphql.instrumentation.enabled` | `true` | Enable query-level timing instrumentation |
| `management.metrics.dgs-graphql.data-loader-instrumentation.enabled` | `true` | Enable data loader metrics |

## Extended Scalars

| Property | Default | Description |
|---|---|---|
| `dgs.graphql.extensions.scalars.time-dates.enabled` | `true` | Enable `DateTime`, `Date`, `Time`, `LocalTime` |
| `dgs.graphql.extensions.scalars.numbers.enabled` | `true` | Enable `Long`, `BigDecimal`, `PositiveInt`, etc. |
| `dgs.graphql.extensions.scalars.objects.enabled` | `true` | Enable `JSON`, `Object`, `URL`, etc. |
| `dgs.graphql.extensions.scalars.locale-currency.enabled` | `true` | Enable `Locale`, `CountryCode`, `Currency` |

## Schema Printing

| Property | Default | Description |
|---|---|---|
| `dgs.graphql.schema-printer.enabled` | `false` | Expose schema at `/schema.graphqls` endpoint |

## Example `application.yml`

```yaml
dgs:
  graphql:
    path: /graphql
    introspection:
      enabled: false   # Disable in production
    schema-printer:
      enabled: false
    virtualthreads:
      enabled: true    # JDK 21+ only

spring:
  graphql:
    websocket:
      path: /subscriptions
  threads:
    virtual:
      enabled: true    # JDK 21+ only

logging:
  level:
    notprivacysafe: OFF   # Suppress graphql-java query logging
```
