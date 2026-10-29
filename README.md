# spring-boot-testing-skills

Agent skills for testing Spring Boot apps (Spring Boot 4 / Spring Framework 7), distilled from the book *Spring Boot Testing In Action* and its samples.

| Skill | Use it for |
|---|---|
| [`spring-testing-strategy`](skills/spring-testing-strategy/SKILL.md) | Choosing the kind of test: unit, `@SpringBootTest`, slice, Modulith module test, `SpringApplicationBuilder`, browser |
| [`spring-testing-fast`](skills/spring-testing-fast/SKILL.md) | Keeping suites fast by avoiding application-context cache thrashing |
| [`spring-testing-web`](skills/spring-testing-web/SKILL.md) | HTTP-level tests with `MockMvcTester`, `RestTestClient`, `WebTestClient`; full-server tests |
| [`spring-testing-browser`](skills/spring-testing-browser/SKILL.md) | HtmlUnit and Selenium tests of rendered pages and JavaScript |
| [`spring-testing-properties`](skills/spring-testing-properties/SKILL.md) | `@ConfigurationProperties` binding, validation and startup-failure tests |
| [`spring-testing-persistence`](skills/spring-testing-persistence/SKILL.md) | Repositories, queries, JPA mappings, constraints, migrations and transactions against Testcontainers; data slices |
| [`spring-testing-http-client`](skills/spring-testing-http-client/SKILL.md) | Outbound HTTP clients (`RestClient`, `@HttpExchange`) with `MockRestServiceServer`, `@RestClientTest` or a stub server |
| [`spring-testing-security`](skills/spring-testing-security/SKILL.md) | Fake users, CSRF, URL and method authorization, OIDC/JWT, security components |

## Installation

Copy (or symlink) the directories under `skills/` into `.claude/skills/` in your project, or into `~/.claude/skills/` for all projects.
