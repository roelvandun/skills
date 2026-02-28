# DGS Schema Reloading Reference

## Overview

DGS caches schema initialization for production efficiency. During development (especially with hot-reload tools like JRebel), you can configure DGS to re-initialize the schema on each request.

## Method 1: `dgs.reload` Property

```yaml
dgs:
  reload: true
```

Enables schema re-initialization on every request. Use only in development.

## Method 2: `laptop` Spring Profile

Activate the built-in development profile:

```yaml
spring:
  profiles:
    active: laptop
```

The `laptop` profile automatically enables development mode, equivalent to `dgs.reload: true`.

## Method 3: Custom `ReloadSchemaIndicator`

For fine-grained control (e.g., reloading only when an external schema source changes):

```java
@Component
public class DatabaseSchemaReloader implements ReloadSchemaIndicator {

    private volatile boolean schemaChanged = false;

    @Override
    public boolean reloadIndicator() {
        boolean reload = schemaChanged;
        schemaChanged = false;  // Reset after reporting
        return reload;
    }

    // Called externally when schema source changes
    public void markDirty() {
        this.schemaChanged = true;
    }
}
```

When `reloadIndicator()` returns `true`, DGS re-runs all `@DgsTypeDefinitionRegistry` and `@DgsCodeRegistry` methods before the next request.

## Dynamic Schema Use Case

Combine with `@DgsTypeDefinitionRegistry` and `@DgsCodeRegistry` for fully dynamic schemas:

```java
@DgsComponent
public class DynamicSchemaProvider {

    @Autowired
    private SchemaRepository schemaRepository;

    @DgsTypeDefinitionRegistry
    public TypeDefinitionRegistry registry(TypeDefinitionRegistry existing) {
        // Load schema from DB / remote source every time this is called
        String schemaSdl = schemaRepository.getCurrentSchema();
        return new SchemaParser().parse(schemaSdl);
    }
}
```

When `ReloadSchemaIndicator` triggers, `@DgsTypeDefinitionRegistry` is re-invoked to build a fresh schema.

## Performance Note

Schema reloading is expensive. The `ReloadSchemaIndicator` approach gives you control over frequency. Never use `dgs.reload: true` in production — use it only for local development.
