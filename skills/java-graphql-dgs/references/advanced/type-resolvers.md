# DGS Type Resolvers for Abstract Types Reference

## Overview

Whenever you use `interface` or `union` types in your GraphQL schema, you must register a type resolver so graphql-java can map Java instances to the correct GraphQL type at runtime.

**Exception:** If your Java class names match your GraphQL type names exactly, DGS creates a type resolver automatically — no code needed.

## Interface Types

Schema:

```graphql
type Query {
    movies: [Movie]
}
interface Movie {
    title: String
}
type ActionMovie implements Movie {
    title: String
    nrOfExplosions: Int
}
type ScaryMovie implements Movie {
    title: String
    scareFactor: Int
}
```

Data fetcher returning mixed types:

```java
@DgsData(parentType = "Query", field = "movies")
public List<Movie> movies() {
    return List.of(
        new ActionMovie("Die Hard", 42),
        new ScaryMovie("The Shining", 9)
    );
}
```

## Registering a Type Resolver

Use `@DgsTypeResolver` with the `name` set to the interface or union type name:

```java
@DgsComponent
public class MovieTypeResolver {

    @DgsTypeResolver(name = "Movie")
    public String resolveMovie(Movie movie) {
        if (movie instanceof ActionMovie) return "ActionMovie";
        if (movie instanceof ScaryMovie) return "ScaryMovie";
        throw new RuntimeException("Unknown type: " + movie.getClass().getName());
    }
}
```

The resolver receives the Java object and returns the **GraphQL type name** as a `String`.

## Union Types

Union types follow the same pattern:

```graphql
union SearchResult = Human | Droid | Starship

type Query {
    search(text: String): [SearchResult]
}
```

```java
@DgsTypeResolver(name = "SearchResult")
public String resolveSearchResult(Object result) {
    if (result instanceof Human) return "Human";
    if (result instanceof Droid) return "Droid";
    if (result instanceof Starship) return "Starship";
    throw new RuntimeException("Unknown SearchResult type");
}
```

## Placement

`@DgsTypeResolver` can be placed in any `@DgsComponent` class — either alongside the data fetcher for the type, or in a separate dedicated class.

## Automatic Type Resolution

When Java type names match GraphQL type names, DGS registers resolvers automatically:

```java
public class ActionMovie { ... }  // Class name == GraphQL type name
public class ScaryMovie { ... }   // Class name == GraphQL type name
// No @DgsTypeResolver needed!
```

This works for both interface implementations and union members.
