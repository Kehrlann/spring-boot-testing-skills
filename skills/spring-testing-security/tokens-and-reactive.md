# OAuth2, OIDC, JWT and reactive security tests

## OIDC / OAuth2 login (MockMvc)

`SecurityMockMvcRequestPostProcessors.oidcLogin()` installs an `OidcUser` authentication for one request, with no identity provider involved.

```java
// default user: fine for "authenticated but not admin" rows
mvc.get().uri("/admin").with(oidcLogin()).exchange();          // expect 403

// shape the ID token and grant the app's roles explicitly
mvc.get().uri("/admin")
    .with(oidcLogin()
        .idToken(token -> {
            token.subject("oidc-subject-id");
            token.claim(StandardClaimNames.EMAIL, "test@example.com");
        })
        .authorities(new SimpleGrantedAuthority("ROLE_ADMIN")))
    .exchange();                                                // expect 200

// the app wraps users in its own OidcUser subtype: pass an instance of it
mvc.post().uri("/api/city")
    .with(oidcLogin().oidcUser(new DemoOidcUser(authorities, idToken, userInfo)))
    .with(csrf())
    .contentType(MediaType.APPLICATION_JSON).content("{ \"id\": 42 }")
    .exchange();
```

- The default authorities are generic (such as `SCOPE_read`), not your app's roles. Set them with `.authorities(...)`.
- `oidcLogin()` bypasses your `OidcUserService`, so the app's authority mapping does not run. Unit-test that mapping separately (see SKILL.md, **Unit-testing security components**).
- Build `OidcIdToken` with `issuedAt` / `expiresAt`, and `OidcUserInfo.builder().subject(..).name(..).email(..)`, when constructing a custom `OidcUser`.
- Plain OAuth2 (non-OIDC) login: `oauth2Login()`, with `.attributes(...)` and `.authorities(...)`.

## Resource server (bearer tokens)

- JWT: `.with(jwt().jwt(j -> j.subject("alice").claim("scope", "read")).authorities(new SimpleGrantedAuthority("SCOPE_read")))`. Like `oidcLogin()`, this bypasses the app's `JwtAuthenticationConverter`. Test the converter directly on a `Jwt` built with `Jwt.withTokenValue("t").header("alg", "none").claim(...).build()`.
- Opaque tokens: `.with(opaqueToken().attributes(a -> a.put("sub", "alice")).authorities(...))`.
- No token at all → `401`; token without the required scope or authority → `403`.

## WebFlux

`WebTestClient` uses configurers from `org.springframework.security.test.web.reactive.server.SecurityMockServerConfigurers`, applied with `mutateWith`:

```java
client.mutateWith(mockUser("alice").roles("ADMIN"))
    .mutateWith(csrf())
    .post().uri("/api/city").bodyValue("{ \"id\": 42 }")
    .exchange()
    .expectStatus().isCreated();
```

- Equivalents: `mockUser()`, `mockAuthentication(auth)`, `mockOidcLogin()`, `mockOAuth2Login()`, `mockJwt()`, `mockOpaqueToken()`, `csrf()`.
- Reactive method security (`@PreAuthorize` on methods returning `Mono` / `Flux`) can be tested by calling the bean under `@WithMockUser` and verifying with `StepVerifier`. Expect the `AccessDeniedException` as an error signal, not a thrown exception.
