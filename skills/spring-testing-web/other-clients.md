# Other Spring web test clients

`MockMvcTester` is the default for new Spring MVC tests. This file covers WebFlux, the alternative clients, and legacy APIs you'll meet in existing suites.

## WebTestClient (WebFlux)

WebFlux apps have no mock MVC stack. Use `WebTestClient`, from `org.springframework.boot:spring-boot-starter-webflux-test` (test scope).

```java
import org.springframework.boot.webtestclient.autoconfigure.AutoConfigureWebTestClient;
import org.springframework.test.web.reactive.server.WebTestClient;
import org.springframework.test.web.reactive.server.assertj.WebTestClientResponse;

@SpringBootTest
@AutoConfigureWebTestClient
class ApiTests {

    @Autowired WebTestClient client;

    @Test
    void selectCity() {
        client.post().uri("/api/city")
            .contentType(MediaType.APPLICATION_JSON)
            .bodyValue("{ \"id\": 42 }")
            .exchange()
            .expectStatus().isCreated();
    }

    @Test
    void indexPage() {
        var response = WebTestClientResponse.from(client.get().uri("/").exchange());
        assertThat(response).hasStatusOk().bodyText().contains("Paris (France)");
    }
}
```

- With the default `MOCK` environment, the client calls the app's `HttpHandler` directly; Netty doesn't start.
- `exchange()` blocks: test methods aren't reactive, and they don't need to be.
- Built-in assertions: `.expectStatus()`, `.expectBody(String.class).value(b -> assertThat(b)…)`, `.expectBody().json(expected)`, `.expectBody().jsonPath("$.length()").isEqualTo(n)`.
- `WebTestClientResponse.from(...)` bridges to the same AssertJ API as `MockMvcTester`. It is useful when you want one assertion style across MVC and WebFlux projects.
- Large bodies exceed the default codec buffer: `client.mutate().codecs(c -> c.defaultCodecs().maxInMemorySize(1024 * 1024)).build()`.
- Real network calls: `WebTestClient.bindToServer().baseUrl("https://…").build()`. This client is built by hand; the autowired one targets the app under test.
- `WebTestClient` in an MVC project needs `spring-boot-starter-webclient-test`. That's unusual; mainly worth it to keep tests intact while migrating from WebFlux.

## RestTestClient (Spring MVC)

Spring Framework 7 / Boot 4. Same fluent API as `WebTestClient`, for the MVC stack. Enable it with `@AutoConfigureRestTestClient` (`org.springframework.boot.resttestclient.autoconfigure`) and inject `RestTestClient`.

- On `MOCK` it goes through mock MVC, just like `MockMvcTester`; choosing between the two is a style choice.
- On `RANDOM_PORT` it sends real HTTP to the running server, with the base URL preset. It's the default client for full-server tests.
- AssertJ bridge: `RestTestClientResponse.from(client.get().uri("/").exchange())`.

## Legacy APIs in existing suites

- **`MockMvc`** (static builders `get(...)`, `status().isOk()`, Hamcrest matchers): `MockMvcTester` wraps it and adds AssertJ and a discoverable fluent API. Existing tests can stay. Write new tests with `MockMvcTester`. `@Autowired MockMvcTester` and `@Autowired MockMvc` both come from `@AutoConfigureMockMvc`.
- **`TestRestTemplate`**: deprecated as of Spring Framework 7.1 / Boot 4.2, to be removed in the next major version. Existing tests may keep it for now. New tests use `RestTestClient`.
- **`@MockBean` / `@SpyBean`** (Boot < 3.4): replaced by `@MockitoBean` / `@MockitoSpyBean` from Spring Framework (`org.springframework.test.context.bean.override.mockito`).
