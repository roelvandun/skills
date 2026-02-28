# DGS Dynamic Schemas Reference

## Programmatic Type Registration (`@DgsTypeDefinitionRegistry`)

Use `@DgsTypeDefinitionRegistry` when you need to add types programmatically at startup (e.g., from a database or external source):

```java
@DgsComponent
public class DynamicSchemaRegistrar {

    @DgsTypeDefinitionRegistry
    public TypeDefinitionRegistry registerTypes(TypeDefinitionRegistry registry) {
        // Add types using SDL parsing
        SchemaParser parser = new SchemaParser();
        TypeDefinitionRegistry dynamic = parser.parse("type DynamicType { name: String }");
        registry.merge(dynamic);
        return registry;
    }
}
```

The method receives the existing `TypeDefinitionRegistry` (from schema files) and can merge additional types.

## Programmatic Data Fetcher Registration (`@DgsCodeRegistry`)

Use `@DgsCodeRegistry` to register data fetchers for dynamically added types:

```java
@DgsComponent
public class DynamicDataFetcherRegistrar {

    @DgsCodeRegistry
    public GraphQLCodeRegistry.Builder registerFetchers(
            GraphQLCodeRegistry.Builder codeRegistry,
            TypeDefinitionRegistry typeRegistry) {

        DataFetcher<String> nameFetcher = dfe -> "Dynamic value";
        codeRegistry.dataFetcher(
            FieldCoordinates.coordinates("DynamicType", "name"),
            nameFetcher
        );
        return codeRegistry;
    }
}
```

## Runtime Schema Reloading

Use `ReloadSchemaIndicator` to control when the schema is reinitialized at runtime:

```java
@Component
public class DatabaseSchemaReloader implements ReloadSchemaIndicator {

    private volatile boolean schemaChanged = false;

    @Override
    public boolean reloadIndicator() {
        // Return true to trigger a schema reload on the next request
        boolean reload = schemaChanged;
        schemaChanged = false;
        return reload;
    }

    // Call this when the schema source changes
    public void markSchemaChanged() {
        this.schemaChanged = true;
    }
}
```

## Development Hot Reload

For development only — reload schema on every request:

```yaml
dgs:
  reload: true
```

Or activate the `laptop` Spring profile.

## Use Cases for Dynamic Schemas

- Schema stored in a database and loaded at startup
- Schemas assembled from micro-schemas owned by different teams
- Multi-tenant applications where each tenant has a slightly different schema
- Federation gateways that compose schemas at runtime
