# Selenium

Dependency: `org.seleniumhq.selenium:selenium-java` (test scope). Selenium drives a real browser through a WebDriver, so the app must serve real HTTP: use `@SpringBootTest(webEnvironment = RANDOM_PORT)` and build absolute URLs from `@LocalServerPort`.

Spring Boot has no opinion on browser setup; you write it. A browser takes over a second to start, so start one per test class in `@BeforeAll`, not per test:

```java
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
class SeleniumTests {

    @LocalServerPort int port;
    static ChromeDriverService driverService;
    static WebDriver driver;

    @BeforeAll
    static void startBrowser() throws IOException {
        driverService = ChromeDriverService.createDefaultService();
        driverService.start();
        var options = new ChromeOptions();
        options.addArguments("--headless"); // required on CI; remove locally to watch the test
        driver = new ChromeDriver(driverService, options);
        driver.manage().timeouts().implicitlyWait(Duration.ofSeconds(1));
    }

    @AfterAll
    static void stopBrowser() {
        driver.quit();
        driverService.stop();
    }

    @Test
    void autocomplete() {
        driver.get("http://localhost:" + port + "/");

        driver.findElement(By.id("citySearch")).sendKeys("jak");

        var items = new WebDriverWait(driver, Duration.ofSeconds(1)) // results render asynchronously
            .until(numberOfElementsToBe(By.cssSelector("#cityResults .autocomplete-item"), 3));
        assertThat(items).extracting(item -> item.getText().trim())
            .containsExactly("Djakotomé (Benin)", "Jakarta (Indonesia)", "Kamirenjaku (Japan)");
    }
}
```

## Rules

- **Any installed browser works**: swap in `GeckoDriverService` (Firefox), `EdgeDriverService` or `SafariDriverService`. Selenium Manager finds the installed browser and downloads a matching driver. Override with `ChromeDriverService.usingDriverExecutable(...)` / `ChromeOptions.setBinary(...)`. Log what it picked with `logging.level.org.openqa.selenium=DEBUG`.
- **Implicit wait covers presence only.** It makes `findElement` retry until the element *exists*, and `findElements` until *at least one* matches. It does not wait for a list to reach its final length, for text to change, or for an existing element to re-render, so asserting on those right after an action is flaky.
- **Wait explicitly on the state you assert**, with `WebDriverWait` and `ExpectedConditions` (static imports): `numberOfElementsToBe(locator, n)`, `textToBePresentInElementLocated(locator, text)`, `attributeToBe(...)`, `invisibilityOfElementLocated(locator)`. Then assert on what the wait returned or on a fresh lookup.
- **Asserting absence**: with an implicit wait, a missing element costs the full timeout. Set `driver.manage().timeouts().implicitlyWait(Duration.ZERO)` for that test, and wait on the condition explicitly:
  ```java
  new FluentWait<>(driver).withTimeout(Duration.ofSeconds(1)).pollingEvery(Duration.ofMillis(100))
      .until(d -> d.findElements(By.cssSelector(".cities-grid > .card")).isEmpty());
  ```
- **Input**: `sendKeys("text")`, `sendKeys(Keys.ARROW_DOWN)`, `sendKeys(Keys.RETURN)`, `element.click()`. Locate with `By.id`, `By.cssSelector`, `By.className`.
- **Shared browser, shared state**: cookies and local storage persist across the tests of a class. Log users out between tests with `driver.manage().deleteAllCookies()`.
- **Same server-side setup as other Spring tests**: `@MockitoBean` external services, reset repositories in `@BeforeEach`, arrange data through services or repositories, then `driver.get(...)`.

## One browser for the whole suite

To share one browser across test classes, expose it as a bean in a top-level `@TestConfiguration` class, and `@Import` it from the suite's shared meta-annotation so every class has the same configuration and shares the context. A nested `static` configuration class is a different class in each test class, so each one starts its own context (load `spring-testing-fast`).

Spring Boot puts every `WebDriver` bean in a special `webDriver` scope that recreates it for each test: the opposite of the goal. Declare singleton scope explicitly:

```java
@TestConfiguration(proxyBeanMethods = false)
public class WebDriverConfiguration {

    @Bean(destroyMethod = "stop")
    ChromeDriverService driverService() throws IOException {
        var service = ChromeDriverService.createDefaultService();
        service.start();
        return service;
    }

    @Bean(destroyMethod = "quit")
    @Scope(BeanDefinition.SCOPE_SINGLETON)
    ChromeDriver webDriver(ChromeDriverService driverService) {
        var options = new ChromeOptions();
        options.addArguments("--headless");
        var driver = new ChromeDriver(driverService, options);
        driver.manage().timeouts().implicitlyWait(Duration.ofSeconds(1));
        return driver;
    }
}
```
