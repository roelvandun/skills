# DGS File Uploads Reference

## Setup

File uploads require the `multipart-spring-graphql` library when using DGS with Spring for GraphQL:

```xml
<dependency>
    <groupId>name.nkonev.multipart-spring-graphql</groupId>
    <artifactId>multipart-spring-graphql</artifactId>
</dependency>
```

## Schema

Declare the built-in `Upload` scalar in your schema:

```graphql
scalar Upload

type Mutation {
    uploadFile(file: Upload!): String
    uploadMultiple(files: [Upload!]!): [String]
}
```

## Data Fetcher

Use `MultipartFile` from Spring as the argument type:

```java
@DgsMutation
public String uploadFile(@InputArgument("file") MultipartFile file) throws IOException {
    byte[] bytes = file.getBytes();
    String filename = file.getOriginalFilename();
    // Process file...
    return "Uploaded: " + filename;
}
```

For multiple files:

```java
@DgsMutation
public List<String> uploadMultiple(@InputArgument("files") List<MultipartFile> files) {
    return files.stream()
        .map(f -> {
            try { return store(f); } catch (IOException e) { throw new RuntimeException(e); }
        })
        .collect(Collectors.toList());
}
```

## Client Request Format

File uploads use the [GraphQL multipart request spec](https://github.com/jaydenseric/graphql-multipart-request-spec):

```bash
curl -X POST http://localhost:8080/graphql \
  -F operations='{"query":"mutation($file: Upload!){uploadFile(file:$file)}","variables":{"file":null}}' \
  -F map='{"0":["variables.file"]}' \
  -F 0=@/path/to/file.txt
```

## Notes

- The `Upload` scalar is automatically registered when using DGS — no `@DgsScalar` needed
- File upload tests require a real HTTP stack (`@SpringBootTest` with `WebEnvironment.RANDOM_PORT`), not `DgsQueryExecutor`
- For large files, consider streaming to object storage rather than holding bytes in memory
