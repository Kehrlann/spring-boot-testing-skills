# Spring Modulith module tests

- `ApplicationModules.of(MyApplication.class).verify()`, in a plain unit test, checks that modules only use each other's public API and have no cycles.
- `@ApplicationModuleTest` is a full `@SpringBootTest` restricted to the module of the test's package.
  - Override the module with `module = "…"`.
  - Pull in another module, such as the one holding the security configuration, with `extraIncludes = "security"`.
  - Stub other modules' API beans with `@MockitoBean`. When a public interface's implementation is internal to its module, write a small test implementation.
- `@ModuleSlicing` on a Boot slice (`@ModuleSlicing @WebMvcTest`) restricts the slice to the module's components, so no `controllers = …` list is needed (Spring Modulith 2.1+).
- Each module test is its own context, like a slice: weigh it the same way (see **Slices** in [SKILL.md](SKILL.md)).
