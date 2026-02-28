# DGS Testing Reference

## Test Slice: `@EnableDgsTest`

`@EnableDgsTest` is the recommended test annotation for data fetcher tests. It bootstraps only the DGS infrastructure (no full HTTP stack), keeping tests fast.

```xml
<!-- Required test dependency -->
<dependency>
  <groupId>com.netflix.graphql.dgs</groupId>
  <artifactId>dgs-starter-test</artifactId>
  <scope>test</scope>
</dependency>
```

### Minimal unit test

```java
@SpringBootTest(classes = {ShowsDataFetcher.class})
@EnableDgsTest
class ShowsDataFetcherTest {

    @Autowired
    DgsQueryExecutor dgsQueryExecutor;

    @MockBean
    ShowService showService;

    @Test
    void shows_returns_filtered_results() {
        when(showService.findAll("Oz")).thenReturn(List.of(new Show("Ozark", 2017)));

        List<String> titles = dgsQueryExecutor.executeAndExtractJsonPath(
            "{ shows(titleFilter: \"Oz\") { title } }",
            "data.shows[*].title"
        );

        assertThat(titles).containsExactly("Ozark");
    }
}
```

Always explicitly list the `classes` in `@SpringBootTest` to avoid loading the entire application context.

## `DgsQueryExecutor` Methods

| Method | Returns | Use when |
|---|---|---|
| `execute(query)` | `ExecutionResult` | Inspecting the full result including errors |
| `executeAndGetDocumentContext(query)` | `DocumentContext` | Multiple extractions from the same result |
| `executeAndExtractJsonPath(query, path)` | `T` | Extracting a single value |
| `executeAndExtractJsonPathAsObject(query, path, Class<T>)` | `T` | Deserializing extracted data to a class |

### Extracting a nested object

```java
CustomerDetailsDTO customer = dgsQueryExecutor.executeAndExtractJsonPathAsObject(
    "{ user(id: \"123\") { id email name } }",
    "data.user",
    CustomerDetailsDTO.class
);
assertThat(customer.getEmail()).isEqualTo("test@example.com");
```

### Checking for errors

```java
ExecutionResult result = dgsQueryExecutor.execute("{ user(id: \"unknown\") { email } }");
assertThat(result.getErrors()).isNotEmpty();
assertThat(result.getErrors().get(0).getMessage()).contains("not found");
```

## MockMvc Integration: `@EnableDgsMockMvcTest`

Use `@EnableDgsMockMvcTest` when the data fetcher accesses HTTP context (e.g., `@RequestHeader`):

```java
@SpringBootTest(classes = {UserDataFetcher.class})
@EnableDgsMockMvcTest
@AutoConfigureMockMvc
class UserDataFetcherMvcTest {

    @Autowired
    MockMvc mockMvc;

    @MockBean
    UserService userService;

    @Test
    void user_with_auth_header() throws Exception {
        when(userService.findById("123")).thenReturn(new User("123", "test@example.com"));

        mockMvc.perform(post("/graphql")
                .contentType(MediaType.APPLICATION_JSON)
                .header("Authorization", "Bearer token")
                .content("{\"query\": \"{ user(id: \\\"123\\\") { email } }\"}"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.data.user.email").value("test@example.com"));
    }
}
```

## Testing with Variables

Pass variables separately instead of interpolating them into the query string:

```java
@Test
void creates_user_with_variables() {
    Map<String, Object> variables = Map.of(
        "input", Map.of("email", "new@example.com", "name", "Alice")
    );

    String id = dgsQueryExecutor.executeAndExtractJsonPath(
        "mutation CreateUser($input: CreateUserInput!) { createUser(input: $input) { id } }",
        "data.createUser.id",
        variables
    );

    assertThat(id).isNotBlank();
}
```

## Integration Tests

For full integration tests that verify the HTTP layer end-to-end, use `@SpringBootTest(webEnvironment = RANDOM_PORT)` with a `TestRestTemplate` or `WebTestClient`, and post JSON to `/graphql`.

Keep these separate from `@EnableDgsTest` unit tests. Name integration test classes with an `IT` suffix so the surefire plugin excludes them and failsafe picks them up.
