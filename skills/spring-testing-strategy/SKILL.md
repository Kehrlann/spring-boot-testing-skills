---
name: spring-testing-strategy
description: Choose the kind of test for Spring Boot code - plain unit test, @SpringBootTest, test slice, Spring Modulith module test, SpringApplicationBuilder, or browser test. Use when about to write tests in a Spring Boot app, or when deciding where a new test belongs.
---

# Spring testing strategy

Spring Boot tests sit in the middle of the testing spectrum. They prove that your code works *together with Spring*: annotations, auto-configuration, properties, `@ExceptionHandler` mappings, query method names, security rules. Each one costs seconds of application startup, so pick the cheapest test that still exercises what can break.

## Steps

1. **Name what can break.** For each behaviour you are about to test, write down the failure you want to catch: a wrong calculation, a wrong HTTP status, a missing bean, a bad property, a forbidden page shown to the wrong user.
2. **Pick the test type** from the decision table: the first row that matches each failure.
3. **Reuse an existing context.** Before giving a Spring test class its own configuration (properties, profiles, mocks, slices), check what the suite already uses: load `spring-testing-fast`.
4. **Load the skill for the chosen type** and write the tests.

Done when every planned test has a type and a reason drawn from the table.

## Decision table

| What can break | Test type | Skill to load |
|---|---|---|
| Business logic with many branches, calculations, input combinations | Plain JUnit unit test: `new` the class, pass collaborators (mocks or hand-written stubs) to the constructor, call methods directly. No Spring, runs in milliseconds. | — |
| A user flow: HTTP request → controller → services → database | `@SpringBootTest` + `@AutoConfigureMockMvc` + `MockMvcTester` | `spring-testing-web` |
| Servlet-container behaviour: error pages, sessions, Tomcat valves | `@SpringBootTest(webEnvironment = RANDOM_PORT)` | `spring-testing-web` |
| HTML structure, forms, clicks, JavaScript | HtmlUnit (mock MVC) or Selenium (real browser) | `spring-testing-browser` |
| Who may call what: URL rules, method security, CSRF, login | The row above that matches, plus Spring Security test support | `spring-testing-security` |
| `@ConfigurationProperties` binding, validation, app refusing to start on bad config | Validator unit tests, or `SpringApplicationBuilder` | `spring-testing-properties` |
| Custom queries, entity mappings, constraints, migrations, transactional behaviour | Repository tests in the shared `@SpringBootTest`, against the real database engine | `spring-testing-persistence` |
| Outbound HTTP client code: the requests the app sends, how it maps responses and errors | `MockRestServiceServer`, alone or with `@RestClientTest` | `spring-testing-http-client` |
| One technical layer of a large, slow-starting app, while iterating fast | Test slice: `@WebMvcTest`, `@DataJpaTest`, `@RestClientTest`, … (see **Slices**) | the layer's skill |
| One business module of a Spring Modulith app | `@ApplicationModuleTest`; read [modules.md](modules.md) | — |
| The app wires up and its properties are valid | The `contextLoads()` test from start.spring.io | — |

## Rules of thumb

- **Flows in Spring tests, branches in unit tests.** A Spring Boot test expresses "when the user does X, Y happens". When reaching one branch needs elaborate GIVEN setup across many beans, extract that logic into a class and unit-test the branches there.
- **Stub external systems.** Replace third-party APIs and services you don't run locally with `@MockitoBean` or a `@TestBean` stub. Live calls make tests slow, flaky and quota-hungry, and they can't simulate failure modes. Leave real calls to end-to-end tests outside the Spring test suite. The client code that talks to those systems gets its own tests: load `spring-testing-http-client`.
- **Verify through behaviour.** Assert on HTTP responses and on services, not on repository tables, so a data-model refactoring doesn't break behaviour tests.
- **Each distinct configuration is another application context.** Load `spring-testing-fast`.

## Slices

A slice loads one technical layer plus the infrastructure needed to test it. Reach for one only when both hold:

- the app is large or slow to start (many beans, Hibernate scanning, Testcontainers, slow startup logic), and
- you want a fast feedback loop on that layer, for example while doing TDD.

Otherwise a `@SpringBootTest` that reuses the suite's cached context is faster overall: each distinct slice is one more context to start, and many thin slices thrash the context cache.

- **One slice annotation per class.** Each slice carries its own `@BootstrapWith`; combining two fails with `found multiple declarations of @BootstrapWith`.
- **If a slice needs many mocks**, the code under test depends on too much: that is an architecture signal.
- How to write each slice lives with its layer: `@WebMvcTest` in `spring-testing-web`, `@DataJpaTest` / `@DataJdbcTest` / `@JdbcTest` in `spring-testing-persistence`, `@RestClientTest` in `spring-testing-http-client`.

## Startup failures

`@SpringBootTest` can't test an app that is supposed to fail on startup: the class errors before any test method runs. Build the app with `SpringApplicationBuilder` instead: load `spring-testing-properties`.
