# DGS Mutations Reference

## Basic Mutation

Mutations use the same `@DgsComponent` + `@DgsMutation` pattern as queries.

```graphql
type Mutation {
  addRating(title: String, stars: Int): Rating
}
type Rating {
  avgStars: Float
}
```

```java
@DgsComponent
public class RatingMutation {

    @DgsMutation
    public Rating addRating(@InputArgument String title, @InputArgument Integer stars) {
        if (stars < 1) throw new IllegalArgumentException("Stars must be 1-5");
        return ratingService.addRating(title, stars);
    }
}
```

## Input Types

Always use `input` types for complex mutation arguments. They are similar to `type` but can only be used as inputs.

```graphql
type Mutation {
  addRating(input: RatingInput): Rating
}
input RatingInput {
  title: String
  stars: Int
}
```

```java
@DgsMutation
public Rating addRating(@InputArgument RatingInput input) {
    return ratingService.addRating(input.getTitle(), input.getStars());
}
```

DGS automatically deserializes the GraphQL `input` map to the Java class. The Java class must have matching camelCase field names.

## Using DataFetchingEnvironment Directly

If you need the raw arguments (e.g., when `@InputArgument` cannot deserialize):

```java
@DgsMutation
public Rating addRating(DataFetchingEnvironment dfe) {
    Map<String, Object> input = dfe.getArgument("input");
    RatingInput ratingInput = new ObjectMapper().convertValue(input, RatingInput.class);
    return ratingService.addRating(ratingInput.getTitle(), ratingInput.getStars());
}
```

Combining both `@InputArgument` and `DataFetchingEnvironment` in the same method is also supported.

## Sparse Updates (Track Which Fields Were Set)

Enable the `trackInputFieldSet` flag in codegen to detect which optional input fields were explicitly included in the mutation:

```groovy
generateJava {
    trackInputFieldSet = true
}
```

Each field is wrapped in `Optional`, with a `has[FieldName]()` method:

```java
@DgsMutation
public String update(@InputArgument EventInput event) {
    if (event.hasName()) { ... }
    if (event.hasLocation()) { ... }
}
```

This is useful for PATCH-style partial updates — only mutate fields that were sent.
