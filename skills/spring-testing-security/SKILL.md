---
name: spring-testing-security
description: Test Spring Security in Spring Boot apps - fake users (@WithMockUser, @WithUserDetails, user(), authentication(), oidcLogin(), jwt()), CSRF, URL and method authorization rules, custom principals, login pages, and security components such as UserDetailsService or OidcUserService. Use when a test calls secured endpoints or services, or when testing who may access what.
---

# Testing application security

Spring Security test support (`org.springframework.boot:spring-boot-starter-security-test`, test scope) fakes an authenticated user by populating the `SecurityContext`. Load `spring-testing-web` alongside it for HTTP tests, and `spring-testing-browser` for login flows.

For OAuth2 / OIDC login, JWT resource servers or WebFlux, also read [tokens-and-reactive.md](tokens-and-reactive.md).

## Steps

1. **List the rules.** Read the `SecurityFilterChain` and the `@PreAuthorize` / `@Secured` annotations. Write down each protected resource and who may reach it.
2. **Write the access matrix.** For each rule, plan one test per row:
   - **anonymous**: no user at all. Expect a `302` redirect to the login page with form login, or `401` for HTTP Basic or a resource server.
   - **authenticated, missing authority**. Expect `403`, or `AccessDeniedException` from a service.
   - **authorized**. Expect success, and assert on the content too.
   - **each authentication mechanism the app supports** (form login, OIDC, …) when roles are derived differently per mechanism.
3. **Pick the fake user** for each test from the table below.
4. **Handle CSRF** on `POST`/`PUT`/`PATCH`/`DELETE` (see **CSRF**).
5. **Unit-test the security components** (user services, authority mapping, principal types) without Spring.

Done when every rule from step 1 has its anonymous, forbidden and authorized rows, and every unsafe request either carries a CSRF token or deliberately tests its absence.

## Choosing the fake user

| Need | Use |
|---|---|
| Any logged-in user, by name and roles | `@WithMockUser(username = "test-user", roles = "ADMIN")` (or `authorities = "SCOPE_read"`). The principal is Spring's `User`. |
| A real user from the app's `UserDetailsService`, with the app's own principal type | `@WithUserDetails("alice")`. Needs the `UserDetailsService` bean in the context and the user to exist. |
| No user, overriding a class-level annotation | `@WithAnonymousUser` |
| A different user for one request | `.with(user("bob").roles("USER"))` |
| An exact `Authentication` with a custom principal | `.with(authentication(new TestingAuthenticationToken(principal, "ignored", authorities)))` |
| An OIDC login | `.with(oidcLogin()…)`; see [tokens-and-reactive.md](tokens-and-reactive.md) |
| An unauthenticated request | No annotation, no post-processor |

The request post-processors come from `SecurityMockMvcRequestPostProcessors` (`user`, `authentication`, `csrf`, `httpBasic`, `oidcLogin`, `jwt`).

- **Scope**: an annotation on the class applies to every test, and a method-level annotation overrides it. A `.with(...)` post-processor overrides both, for that one request. That lets one test act as several users: an admin by annotation, plus `selectCityAs("another-user", …)` through `.with(user("another-user"))`.
- **Identity**: `@WithMockUser(username = "test-user")` makes `authentication.getName()` return `"test-user"`. Assert per-user data with that same identity.
- **Thread-bound**: annotations set the `SecurityContext` on the test thread. They apply to `MockMvcTester` and HtmlUnit on the mock MVC stack, and to direct service calls. A `RANDOM_PORT` server handles requests on its own threads, so there you log in for real (submit the login form, send HTTP Basic credentials, or present a token).

## Custom principal types

When controllers take `@AuthenticationPrincipal MyUser user`, or cast the principal, `@WithMockUser` provides a Spring `User` that is not a `MyUser`. The argument resolves to `null` and the controller fails, typically with a `NullPointerException`. Provide the real type instead:

- `@WithUserDetails("alice")` when the app has a `UserDetailsService` returning `MyUser`.
- `.with(authentication(token))` with a test double. A small builder keeps tests readable:

```java
public class TestUser implements WeatherUser {
    // fields + getters for id, username, email ...

    public static Builder withUsername(String username) { return new Builder().withUsername(username); }

    public static final class Builder {
        private String id = "~~ignored id~~";
        private String username = "Test User";
        private String email = "test@example.com";
        private final Set<String> authorities = new HashSet<>();
        // withId / withUsername / withEmail / withAuthority ... return this

        public Authentication toAuthentication() {
            return new TestingAuthenticationToken(new TestUser(id, username, email),
                    "~~ignored credentials~~", AuthorityUtils.createAuthorityList(authorities));
        }
    }
}

// in a test
var response = mvc.get().uri("/api/me")
    .with(authentication(TestUser.withUsername("Test User").withEmail("test@example.com").toAuthentication()))
    .exchange();
```

For a reusable annotation that builds such a principal, write a custom annotation meta-annotated with `@WithSecurityContext(factory = …)`.

## CSRF

With CSRF protection on (the default), an unsafe request without a token gets `403`, even for an admin. That 403 is easy to misread as an authorization failure.

- **Per request**: `.with(csrf())`.
- **For every request of a context**: register a `MockMvcBuilderCustomizer` in a `@TestConfiguration`. Put it in the suite's shared configuration, since a one-off `@TestConfiguration` costs a new context (load `spring-testing-fast`).
  ```java
  @Bean
  MockMvcBuilderCustomizer defaultCsrf() {
      return builder -> builder.defaultRequest(MockMvcRequestBuilders.get("/").with(csrf()));
  }
  ```
- **Proving protection is on**: send the unsafe request without `csrf()`, or with `csrf().useInvalidToken()`, and expect `403`.
- **Browser tests**: the page's own JavaScript handles the token (for example the SPA cookie → `X-XSRF-TOKEN` header with `csrf(CsrfConfigurer::spa)`), so HtmlUnit and Selenium flows need no extra setup.

## Slices and modules: load the security configuration

A slice or module test that doesn't load your `SecurityFilterChain` runs under Spring Boot's default security instead of your rules: everything requires authentication, with no role rules. Tests then pass or fail for the wrong reasons. Load the real configuration:

- `@WebMvcTest(controllers = …)` + `@Import(SecurityConfiguration.class)`.
- Spring Modulith: `@ModuleSlicing(extraIncludes = "security") @WebMvcTest`, or `@ApplicationModuleTest(extraIncludes = "security")`.
- `@WithUserDetails` needs the `UserDetailsService` bean, which usually lives in that same configuration.

## Method security

Test `@PreAuthorize` rules by calling the bean directly. The security configuration with `@EnableMethodSecurity` must be in the context:

```java
@ApplicationModuleTest(extraIncludes = "security") // or a @SpringBootTest
class CityServiceTests {

    @Autowired CityService cityService;

    @Test
    @WithMockUser(roles = "ADMIN")
    void adminSeesEveryone() { assertThat(cityService.getSelectedCitiesByUser()).containsOnlyKeys(ALICE, BOB); }

    @Test
    @WithMockUser(roles = "USER")
    void forbiddenForNonAdmins() {
        assertThatExceptionOfType(AccessDeniedException.class).isThrownBy(cityService::getSelectedCitiesByUser);
    }

    @Test
    void requiresAuthentication() {
        assertThatExceptionOfType(AuthenticationCredentialsNotFoundException.class)
            .isThrownBy(cityService::getSelectedCitiesByUser);
    }
}
```

## Login pages and conditional security features

- **Assert where anonymous users land**: `assertThat(response).hasStatus(HttpStatus.FOUND).hasRedirectedUrl("/login")`. With HtmlUnit, `assertThat(page.getUrl()).hasPath("/login")` after loading a protected page under `@WithAnonymousUser`.
- **Full login/logout round trip**: use HtmlUnit against real credentials from the app's user store. Type into `#username` and `#password`, click submit, `waitForBackgroundJavaScript`, then assert the landing URL and the welcome text. Load `spring-testing-browser`.
- **Features that switch on with a bean** (an OAuth2 login button shown only when a `ClientRegistrationRepository` exists): test both states. Run one context without the bean, and one with a nested `@TestConfiguration` providing an `InMemoryClientRegistrationRepository`. That is two contexts on purpose.

## Output escaping

Pages that render user-controlled values: stub the data source with markup (`"<script>alert('pwned')</script>"`) and assert the escaped form is present and the raw tag absent:

```java
assertThat(response).bodyText()
    .contains("&lt;script&gt;alert(&#39;pwned&#39;)&lt;/script&gt;")
    .doesNotContain("<script>");
```

## Unit-testing security components

User services, principal types and authority mapping are plain classes: construct them with `new`, no context.

- **`UserDetailsService`**: cover found, `UsernameNotFoundException`, password encoding prefix (`{noop}`, `{bcrypt}`), role prefixing (`USER` → `ROLE_USER`), and `eraseCredentials()` not leaking into the stored user (load twice and compare).
- **Code taking an `Authentication`**: pass `new TestingAuthenticationToken("alice", null)`.
- **Custom `OidcUserService` / authority mapping**: build the request by hand. A `ClientRegistration` without a user-info URI makes `OidcUserService` build the user from the ID token alone, with no HTTP call:
  ```java
  var registration = ClientRegistration.withRegistrationId("dex").clientId("app")
      .authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
      .redirectUri("{baseUrl}/login/oauth2/code/{registrationId}")
      .authorizationUri("https://idp.example.com/authorize").tokenUri("https://idp.example.com/token")
      .scope("openid", "email", "profile").build();
  var idToken = OidcIdToken.withTokenValue("id-token").subject("12345").claim("email", "jane@admin.example.com")
      .issuedAt(Instant.now()).expiresAt(Instant.now().plusSeconds(60)).build();
  var accessToken = new OAuth2AccessToken(OAuth2AccessToken.TokenType.BEARER, "access-token",
      Instant.now(), Instant.now().plusSeconds(60));

  var user = new DemoOidcUserService().loadUser(new OidcUserRequest(registration, accessToken, idToken));
  ```
- **Probe the edges of every mapping rule**: missing claim, look-alike values (`jane@other.admin.example.com` must not become admin when only `admin.example.com` grants it), case differences.
