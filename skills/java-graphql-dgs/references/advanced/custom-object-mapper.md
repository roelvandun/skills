# DGS Custom Object Mapper Reference

## Customizing Jackson ObjectMapper

The DGS framework uses Jackson for deserializing input types and serializing responses. To customize the `ObjectMapper`, define a `Jackson2ObjectMapperBuilder` bean:

```java
@Bean
public Jackson2ObjectMapperBuilder objectMapperBuilder() {
    Jackson2ObjectMapperBuilder builder = new Jackson2ObjectMapperBuilder();
    builder.featuresToEnable(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES);
    builder.featuresToDisable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
    builder.modules(new JavaTimeModule());
    return builder;
}
```

## What This Affects

The custom `ObjectMapper` is used for:
- Deserializing `@InputArgument` annotated parameters
- Deserializing variables from the GraphQL request
- Serializing response data (for types not handled by scalars)

## Scalar Serialization Is NOT Affected

**Important:** The custom `ObjectMapper` does **not** affect how custom scalars serialize or deserialize values. Scalar serialization is handled by the `Coercing` implementation you provide via `@DgsScalar`. If you need custom date formatting for a `DateTime` scalar, implement that in the `Coercing`, not in the `ObjectMapper`.

## Module Registration Example

```java
@Bean
public Jackson2ObjectMapperBuilder objectMapperBuilder() {
    Jackson2ObjectMapperBuilder builder = new Jackson2ObjectMapperBuilder();
    // Register custom modules
    builder.modules(
        new JavaTimeModule(),
        new ParameterNamesModule()
    );
    // Configure serialization
    builder.serializationInclusion(JsonInclude.Include.NON_NULL);
    return builder;
}
```

## Using `ObjectMapper` Directly in Data Fetchers

If you need direct access to the `ObjectMapper` in your data fetcher:

```java
@DgsComponent
public class MyDataFetcher {

    @Autowired
    private ObjectMapper objectMapper;

    @DgsQuery
    public MyType myQuery(DataFetchingEnvironment dfe) {
        Map<String, Object> raw = dfe.getArgument("input");
        return objectMapper.convertValue(raw, MyType.class);
    }
}
```

This is the same `ObjectMapper` bean configured via `Jackson2ObjectMapperBuilder`.
