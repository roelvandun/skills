# DGS Custom Directives Reference

## Overview

Directives allow cross-cutting schema logic (authorization, formatting, caching). DGS supports custom schema directives via `SchemaDirectiveWiring`.

## 1. Declare the Directive in Schema

```graphql
directive @uppercase on FIELD_DEFINITION

type Query {
    hello: String @uppercase
}
```

## 2. Implement `SchemaDirectiveWiring`

```java
@DgsDirective(name = "uppercase")
public class UppercaseDirective implements SchemaDirectiveWiring {

    @Override
    public GraphQLFieldDefinition onField(SchemaDirectiveWiringEnvironment<GraphQLFieldDefinition> env) {
        GraphQLFieldDefinition field = env.getElement();
        DataFetcher<?> originalFetcher = env.getCodeRegistry()
            .getDataFetcher(env.getFieldsContainer(), field);

        // Wrap the original data fetcher
        DataFetcher<?> wrappedFetcher = dfe -> {
            Object result = originalFetcher.get(dfe);
            if (result instanceof String) {
                return ((String) result).toUpperCase();
            }
            return result;
        };

        env.getCodeRegistry().dataFetcher(env.getFieldsContainer(), field, wrappedFetcher);
        return field;
    }
}
```

The `@DgsDirective` annotation on a `@Component` auto-registers the directive wiring.

## 3. Method Hooks

`SchemaDirectiveWiring` offers several hook methods depending on what the directive is applied to:

| Method | Applies to |
|---|---|
| `onField` | `FIELD_DEFINITION` |
| `onObject` | `OBJECT` |
| `onInterface` | `INTERFACE` |
| `onArgument` | `ARGUMENT_DEFINITION` |
| `onInputObject` | `INPUT_OBJECT` |
| `onScalar` | `SCALAR` |
| `onEnum` | `ENUM` |

## Authorization Example

```java
@DgsDirective(name = "auth")
public class AuthDirective implements SchemaDirectiveWiring {

    @Override
    public GraphQLFieldDefinition onField(SchemaDirectiveWiringEnvironment<GraphQLFieldDefinition> env) {
        DataFetcher<?> original = env.getDataFetcher();
        DataFetcher<?> secured = dfe -> {
            // Check authentication before delegating
            if (!isAuthenticated(dfe.getContext())) {
                throw new AccessDeniedException("Not authenticated");
            }
            return original.get(dfe);
        };
        env.getCodeRegistry().dataFetcher(env.getFieldsContainer(), env.getElement(), secured);
        return env.getElement();
    }
}
```

## Manual Registration

If not using `@DgsDirective`, register via `@DgsRuntimeWiring`:

```java
@DgsRuntimeWiring
public RuntimeWiring.Builder addDirectives(RuntimeWiring.Builder builder) {
    return builder.directive("uppercase", new UppercaseDirective());
}
```
