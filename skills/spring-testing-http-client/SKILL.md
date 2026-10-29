---
name: spring-testing-http-client
description: Test outbound HTTP client code in Spring Boot apps - RestClient, HTTP interface clients (@HttpExchange), RestTemplate - with MockRestServiceServer, @RestClientTest, or a stub HTTP server. Use when testing code that calls a third-party or remote API: the request it sends, how it maps responses, error statuses, retries, timeouts.
---

# Testing outbound HTTP clients

Two different things get tested:

- **Code that uses the client** (services, controllers): stub the client bean with `@MockitoBean` in the app's regular tests (load `spring-testing-web`).
- **The client itself**: does it send the right request, map the response, and handle errors? That is this skill.

Keep the client thin: one class per remote API that turns method calls into HTTP requests and responses into domain objects. App tests then mock that one class, and the tests below cover it.

| Need | Technique | Cost |
|---|---|---|
| Request shape, response mapping, error statuses | `MockRestServiceServer.bindTo(builder)` in a plain unit test | milliseconds, no context |
| The same, through Boot's configured `RestClient.Builder` (Jackson settings, `RestClientCustomizer`s, message converters) | `@RestClientTest` | a small slice context |
| Connect/read timeouts, connection failures, SSE, `WebClient` | A stub HTTP server (WireMock, OkHttp `MockWebServer`) | real sockets |

## Steps

1. **List the operations.** For each client method, write down the request (method, path, query parameters, headers, body) and every response it handles: success shapes, empty bodies, each error status.
2. **Pick the technique** from the table.
3. **Per operation**, write one test asserting the request and the mapped result, plus one test per error status the client handles or must propagate.
4. **End every test with `mockServer.verify()`**, so an expected request that was never sent fails the test.

Done when every client operation has its request and its response mapping asserted, each handled error status is simulated, and every test ends in `mockServer.verify()`.

## MockRestServiceServer in a unit test

The client takes a `RestClient.Builder` and builds its `RestClient` from it. Bind the mock server to a builder before handing it over:

```java
import static org.hamcrest.Matchers.startsWith;
import static org.springframework.test.web.client.match.MockRestRequestMatchers.*;
import static org.springframework.test.web.client.response.MockRestResponseCreators.*;

class WeatherClientTests {

    RestClient.Builder builder = RestClient.builder();
    MockRestServiceServer mockServer = MockRestServiceServer.bindTo(builder).build();
    WeatherClient client = new WeatherClient(builder);

    @Test
    void currentWeather() {
        mockServer.expect(requestTo(startsWith("https://api.example.com/weather")))
            .andExpect(method(HttpMethod.GET))
            .andExpect(queryParam("lat", "32.99"))
            .andRespond(withSuccess("""
                { "temperature": 20.0, "wind": 3.5 }
                """, MediaType.APPLICATION_JSON));

        var weather = client.getCurrentWeather(32.99, -96.75);

        assertThat(weather.temperature()).isEqualTo(20.0);
        mockServer.verify();
    }
}
```

A plain `RestClient.builder()` has Spring's default message converters, not Boot's configured ones. When the mapping depends on Boot's Jackson configuration or on `RestClientCustomizer` beans, use `@RestClientTest`.

## `@RestClientTest`

Starter: `org.springframework.boot:spring-boot-starter-restclient-test` (test scope). Package: `org.springframework.boot.restclient.test.autoconfigure`.

```java
@RestClientTest(components = WeatherClient.class)
class WeatherClientTests {

    @Autowired WeatherClient client;
    @Autowired MockRestServiceServer mockServer;

    // tests as above
}
```

- The slice wires Boot's `RestClient.Builder` to the injectable `MockRestServiceServer`.
- **List the components explicitly**: slice filtering reads class metadata and can't discover which beans take a builder.
- **Build from the injected builder.** A component that calls `RestClient.create()` itself bypasses the mock server and makes real calls.
- Provide the component's other dependencies (a properties object, a clock) with `@Import` or `@MockitoBean`.
- Load `spring-testing-strategy` before adding many of these: each distinct `components = …` is one more cached context.

## Expectations and responses

- **Order**: expectations are ordered by default, and each matches one request. For unordered expectations in a unit test: `MockRestServiceServer.bindTo(builder).ignoreExpectOrder(true).build()`.
- **Request matchers**: `requestTo(uri)`, `method(...)`, `queryParam(name, values…)`, `header(name, values…)`, `content().json(expected)` for the request body, `jsonPath("$.city").value("Paris")`.
- **Custom matcher**: a `RequestMatcher` lambda rejects a request by throwing `AssertionError`.
- **Counts**: `mockServer.expect(ExpectedCount.times(3), requestTo(...))`, `ExpectedCount.between(1, 2)`, `ExpectedCount.never()`.
- **Retries**: queue the failure, then the success, as two expectations (`withServerError()` then `withSuccess(...)`), and assert the client returns the successful result.
- **Responses**: `withSuccess(body, mediaType)`, `withStatus(HttpStatus.NOT_FOUND)`, `withServerError()`, `withBadRequest()`. Add a body or headers with `.body(...)` and `.headers(...)`.
- **Errors**: by default `RestClient` throws `HttpClientErrorException` / `HttpServerErrorException` on 4xx/5xx. Assert what the client turns them into (its `onStatus` handlers, a domain exception, an empty result).
- **Limits**: the mock server replaces the request factory, so it can't test connect/read timeouts, connection failures, SSE or websockets. Use a stub HTTP server for those.

## HTTP interface clients

For an `@HttpExchange` interface backed by a `RestClient`, bind the mock server to the builder behind the proxy:

```java
var builder = RestClient.builder().baseUrl("https://api.example.com");
var mockServer = MockRestServiceServer.bindTo(builder).build();
var api = HttpServiceProxyFactory.builderFor(RestClientAdapter.create(builder.build())).build()
    .createClient(WeatherApi.class);
```

## Stub HTTP server

Use one when the behaviour lives in the HTTP stack: timeouts, connection resets, streaming, or a reactive `WebClient`.

- Start the server once for the suite, and reset its stubs in `@BeforeEach`.
- Point the client at it with a `DynamicPropertyRegistrar` bean in the suite's shared test configuration: `registry -> registry.add("weather.base-url", server::baseUrl)`. A `@DynamicPropertySource` method in each test class is part of that class's cache key, so every class gets its own context (load `spring-testing-fast`).
- WireMock: a read timeout is `stubFor(get(urlPathEqualTo("/weather")).willReturn(ok().withFixedDelay(2_000)))`, and a dropped connection is `aResponse().withFault(Fault.CONNECTION_RESET_BY_PEER)`.
