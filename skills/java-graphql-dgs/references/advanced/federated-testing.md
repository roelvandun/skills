# DGS Federated Testing Reference

## Testing `_entities` Queries

Federation subgraphs expose a special `_entities` query that a gateway calls to resolve entity types. Test this query directly with `DgsQueryExecutor`:

```java
@SpringBootTest(classes = {MovieEntityFetcher.class, DgsAutoConfiguration.class})
@EnableDgsTest
class MovieEntityFetcherTest {

    @Autowired DgsQueryExecutor queryExecutor;

    @Test
    void resolveMovieEntity() {
        String result = queryExecutor.executeAndExtractJsonPath(
            "{ _entities(representations: [{__typename: \"Movie\", movieId: \"1\"}]) " +
            "{ ... on Movie { title } } }",
            "$.data._entities[0].title"
        );
        assertThat(result).isEqualTo("Inception");
    }
}
```

## Using Type-Safe Entity Queries (Codegen)

With `generateClient = true` in the codegen plugin, use the generated `EntitiesGraphQLQuery` builder:

```java
@Test
void resolveMovieEntityTypeSafe() {
    EntitiesGraphQLQuery query = new EntitiesGraphQLQuery.Builder()
        .addRepresentationAsVariable(
            MovieRepresentation.newBuilder().movieId("1").build()
        )
        .build();

    EntitiesProjectionRoot<?, ?> projection = new EntitiesProjectionRoot<>()
        .onMovie()
        .title()
        .movieId();

    GraphQLQueryRequest request = new GraphQLQueryRequest(query, projection);

    List<String> titles = queryExecutor.executeAndExtractJsonPath(
        request.serialize(),
        "$.data._entities[*].title"
    );
    assertThat(titles).containsExactly("Inception");
}
```

## Testing `@extends` Types

When testing a type that extends another service's type, supply a mock or stub for the `@external` fields since they come from the gateway:

```java
// The Movie type has @external movieId — simulate the gateway passing it
String query = "{ _entities(representations: [{__typename: \"Movie\", movieId: \"1\"}]) " +
               "{ ... on Movie { reviews { text } } } }";
```

The entity fetcher receives the `movieId` in the `Map<String, Object> values` argument.

## Verifying `@DgsEntityFetcher` Registration

If `@DgsEntityFetcher(name = "Movie")` is not found, the `_entities` query returns `null` for that entity. Add a sanity test to catch missing registrations:

```java
@Test
void entityFetcherRegistered() {
    assertThat(queryExecutor.executeAndExtractJsonPath(
        "{ _entities(representations: [{__typename: \"Movie\", movieId: \"99\"}]) " +
        "{ __typename } }",
        "$.data._entities[0].__typename"
    )).isEqualTo("Movie");
}
```
