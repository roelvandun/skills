# DGS Scalars Reference

## Custom Scalar with `@DgsScalar`

Implement `Coercing` and annotate with `@DgsScalar`. Also declare the scalar in your schema.

```java
@DgsScalar(name = "DateTime")
public class DateTimeScalar implements Coercing<LocalDateTime, String> {

    @Override
    public String serialize(Object dataFetcherResult) throws CoercingSerializeException {
        if (dataFetcherResult instanceof LocalDateTime) {
            return ((LocalDateTime) dataFetcherResult).format(DateTimeFormatter.ISO_DATE_TIME);
        }
        throw new CoercingSerializeException("Not a valid DateTime");
    }

    @Override
    public LocalDateTime parseValue(Object input) throws CoercingParseValueException {
        return LocalDateTime.parse(input.toString(), DateTimeFormatter.ISO_DATE_TIME);
    }

    @Override
    public LocalDateTime parseLiteral(Object input) throws CoercingParseLiteralException {
        if (input instanceof StringValue) {
            return LocalDateTime.parse(((StringValue) input).getValue(), DateTimeFormatter.ISO_DATE_TIME);
        }
        throw new CoercingParseLiteralException("Value is not a valid ISO date time");
    }
}
```

Schema:

```graphql
scalar DateTime
```

## Using Extended Scalars (Auto-registration)

Add the `graphql-dgs-extended-scalars` module (no version needed if using the BOM) and declare the scalar in schema:

```xml
<dependency>
  <groupId>com.netflix.graphql.dgs</groupId>
  <artifactId>graphql-dgs-extended-scalars</artifactId>
</dependency>
```

Extended scalars auto-register: `DateTime`, `Date`, `Time`, `LocalTime`, `UUID`, `Long`, `BigDecimal`, `PositiveInt`, `URL`, `JSON`, `Locale`, `CountryCode`, `Currency`, and more.

Disable groups selectively:

```yaml
dgs.graphql.extensions.scalars:
  time-dates.enabled: false
  numbers.enabled: false
  objects.enabled: false
```

## Manual Registration via `@DgsRuntimeWiring`

```java
@DgsComponent
public class LongScalarRegistration {

    @DgsRuntimeWiring
    public RuntimeWiring.Builder addScalar(RuntimeWiring.Builder builder) {
        return builder.scalar(Scalars.GraphQLLong);
    }
}
```

## Codegen Type Mapping for Extended Scalars

When using the Gradle codegen plugin, add explicit type mappings:

```groovy
generateJava {
    typeMapping = [
        "DateTime"    : "java.time.LocalDateTime",
        "Url"         : "java.net.URL",
        "PositiveInt" : "java.lang.Integer"
    ]
}
```

## Testing with Extended Scalars

Include `DgsExtendedScalarsAutoConfiguration` in `@SpringBootTest`:

```java
@SpringBootTest(classes = {DgsAutoConfiguration.class, DgsExtendedScalarsAutoConfiguration.class, MyDataFetcher.class})
@EnableDgsTest
class MyTest { ... }
```
