---
name: spring-testing-properties
description: Test Spring Boot configuration - @ConfigurationProperties binding and defaults, Bean Validation constraints on properties, and apps that must refuse to start on invalid config. Use when adding or changing configuration properties, their validation, or startup checks such as InitializingBean on configuration.
---

# Testing configuration and properties

Configuration code runs once, at startup, and a single bad value can stop the app. Spring Boot turns configuration into beans in four stages:

**load** property sources → **bind** them onto the `@ConfigurationProperties` object (applying `@DefaultValue`) → **validate** constraints (`@Validated`, `@Valid`, `@Min`, …) → **post-initialize** (`InitializingBean#afterPropertiesSet`, `@PostConstruct`).

Pick the cheapest technique that covers the stage you are testing:

| Stage under test | Technique | Cost |
|---|---|---|
| Validation rules on a properties class | Validator unit test | milliseconds, no context |
| Validation through nested objects (`@Valid` cascade) | Validator unit test on the root object, built from YAML | milliseconds |
| Binding, defaults, post-init checks, the app refusing to start | `SpringApplicationBuilder` with a minimal context | tens of ms per test, no cache |
| The app behaving correctly under a given configuration | `@SpringBootTest` with properties | a cached context per configuration |

Every distinct property set in `@SpringBootTest` is a separate cached context (load `spring-testing-fast`). Keep annotated tests to a default configuration plus at most one "custom" configuration that sets every value you want to check. Test all other variations with the first three techniques.

## Steps

1. **Inventory each properties class.** List its constraints, its `@Valid` cascades into nested objects, its `@DefaultValue`s and relaxed-binding names, and its post-initialization checks.
2. **Validator unit tests for the constraints.** One test per constraint with a violating value, asserting the property path and message. Use boundary values (`-90` passes, `-91` fails). For each `@Valid` cascade, add one test that validates the root object.
3. **`SpringApplicationBuilder` tests for binding and startup.** One successful bind that asserts every default. One failing start per post-initialization check, asserting the root cause and its message.
4. **`@SpringBootTest` only for behaviour that depends on configuration**, reusing the suite's default or single custom configuration.

Done when every constraint has a failing-value test, every `@Valid` cascade is proven from the root object, every `@DefaultValue` is asserted, and every startup check has a failing-start test.

## Validator unit tests

Plain unit tests. No Spring context, so no annotation on the class:

```java
class PreferencesPropertiesTests {

    LocalValidatorFactoryBean validator = new LocalValidatorFactoryBean();

    @BeforeEach
    void setUp() {
        validator.afterPropertiesSet(); // you manage the lifecycle yourself
    }

    @Test
    void coldThresholdTooLow() {
        var violations = validator.validate(new TemperatureThreshold(-100, 40));

        assertThat(violations).singleElement().satisfies(v -> {
            assertThat(v.getPropertyPath()).hasToString("cold");
            assertThat(v.getMessage()).isEqualTo("must be greater than or equal to -90");
        });
    }
}
```

- Validating only the nested object (`TemperatureThreshold`) proves its constraints, but misses a forgotten `@Valid` on the parent field. Validate the root properties object too, and assert on the nested path (`temperatureThreshold.cold`).
- Deeply nested properties make constructor calls unreadable (`new PreferencesProperties(null, null, new …)`). Write the input as YAML instead and map it with Jackson 3. The test dependency is `tools.jackson.dataformat:jackson-dataformat-yaml`:

  ```java
  YAMLMapper mapper = YAMLMapper.builder()
      .propertyNamingStrategy(PropertyNamingStrategies.KEBAB_CASE)
      .build();

  var props = mapper.readValue("""
      temperature-threshold:
        cold: -100
        hot: 100
      """, PreferencesProperties.class);
  assertThat(validator.validate(props)).hasSize(2);
  ```

- Limits: Jackson is not Spring Boot's binder. It applies no `@DefaultValue`, no relaxed binding, and no post-initialization. Cover those with `SpringApplicationBuilder`.

## SpringApplicationBuilder: binding, defaults and startup failure

`@SpringBootTest` can't test a context that fails to start: the class errors before any test runs. Build and run a minimal app yourself:

```java
@Test
void bindsDefaults() {
    var builder = new SpringApplicationBuilder(PreferencesConfiguration.class)
        .properties("preferences.temperature-threshold.cold=5")
        .web(WebApplicationType.NONE);
    try (var context = builder.run()) { // closes the context when the test ends
        var props = context.getBean(PreferencesProperties.class);
        assertThat(props.getTemperatureThreshold().cold()).isEqualTo(5);
        assertThat(props.getTemperatureThreshold().hot()).isEqualTo(25); // default
    }
}

@Test
void rejectsInvertedRange() {
    var builder = new SpringApplicationBuilder(PreferencesConfiguration.class)
        .properties("preferences.temperature-threshold.hot=10",
                    "preferences.temperature-threshold.cold=20")
        .web(WebApplicationType.NONE);

    assertThatThrownBy(builder::run)
        .isInstanceOf(BeanCreationException.class)
        .rootCause()
        .isInstanceOf(IllegalArgumentException.class)
        .hasMessage("preferences.temperature-threshold.hot (10.0) must be higher than preferences.temperature-threshold.cold (20.0)");
}
```

- **Always set `.web(WebApplicationType.NONE)`.** The default is `SERVLET`, which needs a real servlet container. Unlike `@SpringBootTest`, there is no `MOCK` mode.
- **Use the smallest entry point**: the configuration class that declares `@EnableConfigurationProperties(...)` for the properties under test, plus whatever it needs. If the builder needs many classes to start, either the configuration is tangled or the test belongs in `@SpringBootTest`.
- A failed `run()` closes the context itself; a successful one goes in try-with-resources.
- Constraint failures surface as a root cause `BindValidationException`; its message contains the constraint text (`must be greater than or equal to -90`).
- These contexts bypass the context cache, so each test pays its own (small) startup.

### YAML input for the builder

`.properties(...)` takes `key=value` strings or a `java.util.Properties`. Build `Properties` from YAML with these helpers, from a string or a classpath file:

```java
static Properties propertiesFromYaml(String yaml) {
    var factory = new YamlPropertiesFactoryBean();
    factory.setResources(new ByteArrayResource(yaml.getBytes(StandardCharsets.UTF_8)));
    return factory.getObject();
}

static Properties propertiesFromYamlFile(String classpathFile) {
    var factory = new YamlPropertiesFactoryBean();
    factory.setResources(new ClassPathResource("/" + classpathFile));
    return factory.getObject();
}
```

Chain several `.properties(...)` calls to layer file defaults under per-test overrides; the last value declared for a key wins. To supply a whole environment instead, load YAML with `new YamlPropertySourceLoader().load(name, resource)`, add the source to a `StandardEnvironment`, and pass it to `.environment(env)`.

## Feeding properties to annotated tests

For the `@SpringBootTest` row of the table:

- **Inline**: `@SpringBootTest(properties = { "preferences.defaults.dark-mode=true", "preferences.defaults.units=imperial" })`, or a single text block with one property per line. Properties format only; YAML is not supported here.
- **Files**: `@TestPropertySource(locations = { "classpath:units-imperial.properties", "classpath:dark-mode.properties" })`. Files are read in order and later files win. Properties format unless you supply `factory = YamlPropertySourceFactory.class`, a `PropertySourceFactory` that returns `new YamlPropertySourceLoader().load(name, resource.getResource()).get(0)`.
- **Profiles**: `@ActiveProfiles({ "dark-mode", "date-added" })` loads `application-<profile>.yaml` or `.properties` from `src/test/resources`, and supports YAML natively. Also good for switching on a real production feature profile in a subset of tests.
- Keep test-only property files under `src/test/resources` with distinct names. A `src/test/resources/application.properties` *replaces* the main one, so tests silently drift from production config. Override individual values with the mechanisms above instead.
- For a narrow binding test, `@SpringBootTest(classes = PreferencesConfiguration.class, webEnvironment = NONE)` avoids loading the data layer.
- `@DynamicPropertySource` / `DynamicPropertyRegistrar` inject values only known at runtime, such as a Testcontainers port.
