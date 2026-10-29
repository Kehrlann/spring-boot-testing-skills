---
name: spring-testing-browser
description: Browser-level tests of Spring Boot web UIs - HtmlUnit on the mock MVC stack, or Selenium driving a real browser against a RANDOM_PORT server. Use when testing rendered HTML structure, forms, clicks, keyboard input, JavaScript behaviour or login pages, rather than raw HTTP responses.
---

# Browser-level testing

`MockMvcTester` handles HTTP and JSON well, but HTML only as one big string. To check page structure or simulate a user, drive a browser:

| Choose | When | Cost |
|---|---|---|
| **HtmlUnit** | Server-rendered pages, forms, light JavaScript (ES6-era) | JVM-only, runs on mock MVC, no server |
| **Selenium** | Modern JavaScript (`async`/`await`, classes, recent browser APIs such as `PublicKeyCredential` for passkeys), JS-heavy pages | Real browser + driver, full server, slower |

Start with HtmlUnit; move to Selenium when HtmlUnit can't run the page's JavaScript. For a rich SPA built with a JS framework (React, Angular, Vue), the frontend team's tools (Playwright, Cypress) usually fit better than either.

For Selenium, read [selenium.md](selenium.md).

## Steps

1. **Pick the driver** from the table, per page: HtmlUnit unless its JavaScript engine can't run the page.
2. **Arrange server-side state** as for HTTP tests: stub external services, reset data, insert what the page should show.
3. **For each user interaction**, load the page, act (type, click, press a key), wait for the JavaScript to settle, then assert the resulting DOM through stable selectors (ids, `data-role`, semantic classes).
4. **Cover navigation outcomes**: where a form submit, a login or a logout lands, and what an unauthorized user sees.

Done when every interaction under test asserts the DOM state it produces, after waiting for the JavaScript it triggers (`waitForBackgroundJavaScript` in HtmlUnit, an explicit wait on that state in Selenium), and every navigating action asserts the landing URL.

## HtmlUnit

Dependency: `org.htmlunit:htmlunit` (test scope). With `@AutoConfigureMockMvc` (or `@WebMvcTest`), Spring Boot auto-configures an HtmlUnit `WebClient` bound to mock MVC. Request paths need no host or port.

**Import `org.htmlunit.WebClient`**: it is not Spring's reactive `WebClient`.

```java
import org.htmlunit.WebClient;
import org.htmlunit.html.HtmlPage;

@SpringBootTest
@AutoConfigureMockMvc
class HtmlUnitTests {

    @Autowired WebClient webClient;

    @BeforeEach
    void setUp() {
        webClient.getOptions().setFetchPolyfillEnabled(true); // page JS uses fetch()
    }

    @Test
    void addCityWithMouse() throws IOException {
        HtmlPage page = webClient.getPage("/");

        page.<HtmlInput>querySelector("input#citySearch").type("bogot");
        webClient.waitForBackgroundJavaScript(1000); // autocomplete XHR

        page.<HtmlElement>querySelector(".autocomplete-item").click();
        webClient.waitForBackgroundJavaScript(1000); // selection XHR + re-render

        assertThat(page.querySelectorAll(".cities-grid .card"))
            .singleElement()
            .extracting(DomNode::getTextContent).asString()
            .contains("Bogotá (Colombia)");
    }
}
```

- **Find elements** with `querySelectorAll(css)` (returns `DomNode`s) and `querySelector(css)`. Type the result for the action you need: `HtmlInput.type(...)`, `HtmlElement.click()`, `HtmlButton.click()`. Also `getElementById(id)`.
- **Assert on text** with `DomNode::getTextContent`. Select with stable hooks such as ids, `data-role` attributes and semantic classes, and assert on several cards or rows with `.map(...)`.
- **Keys**: `input.type(KeyboardEvent.DOM_VK_RETURN)`, `DOM_VK_DOWN`.
- **After every action that triggers JavaScript, call `webClient.waitForBackgroundJavaScript(timeoutMs)`** before asserting. JS runs asynchronously, so an assertion made too early sees the old DOM.
- **Navigation**: after a click that navigates (form submit, logout), read the new page with `page.getEnclosingWindow().getEnclosedPage()`; assert the URL with `assertThat(page.getUrl()).hasPath("/login")`.
- **JavaScript limits**: HtmlUnit runs JS in a Java engine (a fork of Mozilla Rhino), which lacks modern syntax. A failure looks like a cryptic parse error, for example `missing ; before statement (…/modern.js#11)` on an `async` line. Pin it down with a test that expects the `ScriptException`, or switch to Selenium.
- Server-side setup is the same as for HTTP tests (load `spring-testing-web`): stub external services with `@MockitoBean`, reset data in `@BeforeEach`, arrange data through services or repositories, then load the page.
- Secured pages: the `@WithMockUser` / `@WithUserDetails` annotations apply to HtmlUnit requests too; load `spring-testing-security`.
