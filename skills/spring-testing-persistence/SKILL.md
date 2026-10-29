---
name: spring-testing-persistence
description: Test the persistence layer of Spring Boot apps - Spring Data repositories, derived and custom queries, JPA mappings, database constraints, schema migrations and transactions - with Testcontainers and @ServiceConnection, @DataJpaTest/@DataJdbcTest/@JdbcTest, @Sql and TestEntityManager. Use when adding or changing repositories, queries, entities or migrations, or when Spring tests need a database.
---

# Testing persistence

Test what Spring Data builds from *your* declarations: derived query method names, `@Query` JPQL or SQL, custom repository implementations, entity mappings (relationships, cascades, converters, embeddables), database constraints, and migrations. Plain `save` / `findById` is Spring Data's own code; flow tests exercise it.

## Steps

1. **Inventory** the repositories and entities you touched: queries, mappings, constraints, migrations.
2. **Run against the production engine** with a Testcontainers database shared by the whole suite (below).
3. **Write repository tests in the suite's shared `@SpringBootTest`**: autowire the repositories, reset data in `@BeforeEach`. Use a data slice only in a large, slow-starting app (load `spring-testing-strategy`).
4. **Per query**: data that must match, and look-alike data that must not (boundary dates, case differences, nulls, another user's rows). Assert ordering and paging when the query declares them.
5. **Per mapping**: save, then reload from the database (see **Round trips**) and assert every mapped field and association.
6. **Per constraint**: violate it and expect `DataIntegrityViolationException`.

Done when every derived or custom query has a matching and a non-matching case, every mapping is asserted after a reload from the database, every constraint has a violation test, and the suite's context starts against the production engine with its migrations applied.

## One database container for the whole suite

Dependencies (test scope): `org.springframework.boot:spring-boot-testcontainers` and the Testcontainers module for the engine, such as `org.testcontainers:testcontainers-postgresql` (Testcontainers 2).

```java
import org.testcontainers.postgresql.PostgreSQLContainer;

@TestConfiguration(proxyBeanMethods = false)
public class TestcontainersConfiguration {

    @Bean
    @ServiceConnection
    PostgreSQLContainer postgres() {
        return new PostgreSQLContainer("postgres:17-alpine");
    }
}
```

- `@ServiceConnection` points the datasource at the container; no URL or credential properties needed. The container starts with the context and stops when the context closes.
- `@Import(TestcontainersConfiguration.class)` from the suite's shared meta-annotation, so every class shares one context and one container (load `spring-testing-fast`).
- Starting the context runs Flyway or Liquibase against the container, so `contextLoads()` proves the migrations apply to the real engine.
- **Tie the container's lifecycle to the context, as above.** A `static @Container` field under `@Testcontainers` is stopped by JUnit after its class, while the cached context still points at it: the next class that reuses that context fails with connection errors. Its `@DynamicPropertySource` method also gives each class its own cache key.
- To run the app locally against the same container, add a `main` in `src/test`: `SpringApplication.from(MyApplication::main).with(TestcontainersConfiguration.class).run(args)`.

## Repository tests

```java
@AppTest // the suite's shared meta-annotation, importing TestcontainersConfiguration
class CityRepositoryTests {

    @Autowired CityRepository cityRepository;

    @BeforeEach
    void setUp() {
        cityRepository.deleteAll();
    }

    @Test
    void findsByNamePrefixIgnoringCase() {
        cityRepository.saveAll(List.of(city("Paris"), city("Parma"), city("Lyon")));

        var cities = cityRepository.findByNameStartingWithIgnoreCase("par");

        assertThat(cities).extracting(City::getName).containsExactlyInAnyOrder("Paris", "Parma");
    }
}
```

- **Reset order**: with foreign keys, delete children before parents, or `JdbcTestUtils.deleteFromTables(jdbcTemplate, "selection", "city")` in child-to-parent order.
- **SQL fixtures**: `@Sql("/fixtures/cities.sql")` on the class or a method runs before each test; `executionPhase = AFTER_TEST_METHOD` for cleanup scripts. `@Sql` doesn't change the cache key.
- Repository tests assert on repositories; behaviour tests elsewhere assert through HTTP and services.

## Round trips

JPA's persistence context hands back the object you saved, not what the database holds, so a test that saves and reads in one transaction proves nothing about the mapping.

- **Without a test transaction** (the shared `@SpringBootTest` default), each repository call commits on its own, and `findById` after `save` reads from the database. Lazy associations then throw `LazyInitializationException` outside a transaction, as they would in production: read them inside a `TransactionTemplate`, or through the fetch-join query the app uses.
- **Inside a test transaction** (`@DataJpaTest`, or a `@Transactional` test), flush and clear before reading: `TestEntityManager.persistFlushFind(entity)`, or `entityManager.flush(); entityManager.clear();`.
- Constraints fire when SQL runs. In a transaction, use `saveAndFlush(...)` so the violation surfaces inside the test.

## Transactions in tests

`@Transactional` on a test class or method rolls back after each test. Keep it to repository-level tests. In flow tests it changes what runs:

- With `MockMvcTester`, the controller runs on the test thread inside the test's transaction: lazy loading that fails in production succeeds, and commit-time failures never happen.
- With `RANDOM_PORT` or `@Async` code, other threads can't see the test's uncommitted data, and the rollback doesn't undo what they wrote.

For flow tests, reset data in `@BeforeEach` instead. To test a `@Transactional` service's rollback, keep the test non-transactional: make a collaborator throw (`doThrow` on a `@MockitoBean`, or a `@MockitoSpyBean`), call the service, then assert the repository is unchanged.

## Slices: `@DataJpaTest`, `@DataJdbcTest`, `@JdbcTest`

Only for a large, slow-starting app (load `spring-testing-strategy`). If the suite already has `@SpringBootTest` classes, autowire repositories there instead: you reuse the cached context and pay no second startup.

- `@DataJpaTest` (`org.springframework.boot.data.jpa.test.autoconfigure`, starter `spring-boot-starter-data-jpa-test`) loads entities, repositories, the datasource, migrations and a `TestEntityManager`. `@DataJdbcTest` does the same for Spring Data JDBC, `@JdbcTest` for `JdbcTemplate`.
- Every test is `@Transactional` and rolled back. Opt out with `@Transactional(propagation = Propagation.NOT_SUPPORTED)` on the class or method.
- `@AutoConfigureTestDatabase` (default `replace = NON_TEST`) swaps the datasource for an embedded H2, HSQLDB or Derby *unless* it already points at a test database: a `@ServiceConnection` container, a `@DynamicPropertySource` URL, or a Testcontainers JDBC URL. `@Import(TestcontainersConfiguration.class)` therefore keeps the real engine. Without one, the embedded database must be on the test classpath.
- An embedded database can't run engine-specific SQL: native queries, and migrations that use the engine's types or functions.
- `properties = …` and each slice annotation change the cache key, as on `@SpringBootTest`.
