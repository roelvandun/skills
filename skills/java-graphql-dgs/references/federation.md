# DGS Federation Reference

DGS fully supports **Apollo Federation** for composing multiple GraphQL schemas into a single gateway.

## Defining an Entity Type

Use `@key` directive to define the primary key of a federated entity:

```graphql
type Movie @key(fields: "movieId") {
    movieId: ID!
    title: String
}
```

The `@key` fields uniquely identify an instance of the type across services.

## Entity Fetcher

Implement `@DgsEntityFetcher` to allow the gateway to resolve entities by their key:

```java
@DgsComponent
public class MovieEntityFetcher {

    @DgsEntityFetcher(name = "Movie")
    public Movie movie(Map<String, Object> values) {
        // The gateway calls this with the key fields
        return movieService.findById((String) values.get("movieId"));
    }
}
```

The `name` must exactly match the GraphQL type name.

## Extending a Type from Another Service

Use `@extends` and `@external` to refer to a type owned by another service:

```graphql
type Movie @key(fields: "movieId") @extends {
    movieId: ID! @external
    reviews: [Review]
}
```

Add a normal data fetcher for the extended field — `@DgsEntityFetcher` is only needed if this service needs to contribute its own key.

## External References and External Resolvers

When DGS is the subgraph (not the router), the `Movie` type extended above is "owned" elsewhere. The DGS service only needs to supply a data fetcher for `reviews`:

```java
@DgsComponent
public class ReviewDataFetcher {

    @DgsData(parentType = "Movie", field = "reviews")
    public List<Review> reviews(DgsDataFetchingEnvironment dfe) {
        Movie movie = dfe.getSource();  // Only has movieId (external fields)
        return reviewService.findByMovieId(movie.getMovieId());
    }
}
```

## Key Field Types

Keys can be a single field or multiple fields:

```graphql
type Movie @key(fields: "movieId title") { ... }
```

## Direct Testing of `_entities` Queries

You can test federation entity fetchers directly:

```java
String query = "{ _entities(representations: [{__typename: \"Movie\", movieId: \"1\"}]) { ... on Movie { title } } }";
Map<String, Object> data = queryExecutor.executeAndGetDocumentContext(query)
    .read("data._entities[0]", Map.class);
```

Or using the generated client:

```java
EntitiesGraphQLQuery entitiesQuery = new EntitiesGraphQLQuery.Builder()
    .addRepresentationAsVariable(MovieRepresentation.newBuilder().movieId("1").build())
    .build();
```

## Maven Dependency

No additional dependency needed — federation support is included in `dgs-starter`. Add the schema directives in your `*.graphqls` file; the `@key`, `@extends`, `@external` directives are defined automatically.
