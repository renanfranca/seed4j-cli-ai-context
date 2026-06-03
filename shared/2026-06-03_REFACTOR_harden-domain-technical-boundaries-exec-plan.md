# Harden Domain Technical Boundaries

This ExecPlan is a living document. Keep `Progress`, `Decisions`, `Risks`, and `Lessons Learned` up to date as work advances.

Safety boundary: This task is limited to authorized, defensive maintenance of this repository. Do not provide offensive guidance or policy-bypassing instructions.

## Purpose / Big Picture

The bootstrap domain currently contains technical I/O, JAR inspection, filesystem cache handling, logging framework access, and child JVM command construction. This refactor hardens the architecture so `HexagonalArchTest` catches those leaks and the implementation moves them to `bootstrap/infrastructure/secondary`. Observable success is that the architecture test fails first, then passes after behavior-preserving refactors, while runtime extension launch behavior stays unchanged.

Implementation must use the `tdd-strict-autonomous-quiet` skill: one observable behavior per red-green-refactor cycle, quiet output, focused suite every cycle, and a public-path checkpoint at least every two cycles.

## Scope

In scope:

- Harden `src/test/java/com/seed4j/cli/HexagonalArchTest.java`.
- Remove direct technical dependencies from `src/main/java/com/seed4j/cli/bootstrap/domain`.
- Move JAR/filesystem/process/logging implementation details to `src/main/java/com/seed4j/cli/bootstrap/infrastructure/secondary`.
- Preserve CLI runtime launch behavior, extension runtime loading, system properties, error messages, and exit codes.
- Move or rewrite tests so domain tests cover pure decisions and secondary tests cover technical adapters.

Out of scope:

- Removing all `Path` usage from domain value objects and ports.
- Refactoring `System.err` output from bootstrap primary flow.
- Running `./mvnw clean verify` automatically.

## Definitions

`domain` means business concepts, decisions, pure services, and ports under `..domain..`.

`secondary adapter` means infrastructure that implements ports and touches filesystem, JARs, process execution, YAML, Spring, logging frameworks, or current runtime state.

`runtime extension` means an installed Spring Boot fat JAR used to extend the CLI at runtime.

`overlay cache` means extracted `BOOT-INF/classes` content stored under `~/.config/seed4j-cli/runtime/cache`.

`loader.path` means the Spring Boot `PropertiesLauncher` classpath extension property used to load overlay classes and selected nested libraries.

## Existing Context

`HexagonalArchTest` currently allows broad `java..`, `org.slf4j..`, and `ch.qos.logback.classic..` dependencies in domain, so it does not fail when domain classes perform I/O or use logging framework types.

Known leaks:

- `RuntimeExtensionOverlayCache` uses `Files`, `JarFile`, `InputStream`, `StandardCopyOption`, staging directories, and cleanup.
- `RuntimeExtensionLoaderPathResolver` opens JARs, reads nested JARs, parses `pom.properties`, logs decisions, and builds `loader.path`.
- `RuntimeExtensionStartClassResolver` opens the extension JAR and reads manifest `Start-Class`.
- `RuntimeExtensionCacheIdentityResolver` reads the extension JAR bytes and computes SHA-256.
- `Seed4JCliLauncher` uses `Files.isRegularFile`, Logback APIs, and Spring Boot `PropertiesLauncher` constants.
- `JavaProcessChildLauncher` builds a Java command in domain through `ProcessCommandExecutor`.

## Desired End State

`HexagonalArchTest` rejects technical dependencies in domain while still allowing simple JDK value types such as `Path`, collections, regex, time, and primitives where already used as domain values.

`bootstrap/domain` keeps pure runtime decisions:

- runtime mode selection orchestration
- library identity/version comparison
- missing library selection
- domain records and ports

`bootstrap/infrastructure/secondary` owns:

- filesystem/JAR/manifest/cache/hash operations
- child JVM command construction
- Spring Boot `PropertiesLauncher` constants and system property names
- Logback/SLF4J diagnostic output

## Milestones

### Milestone 1 - Make Architecture Test Fail For Technical Domain Dependencies

#### Goal

Create the intentional RED by tightening the architecture rule.

#### Changes

- [ ] Edit `src/test/java/com/seed4j/cli/HexagonalArchTest.java`.
- [ ] Replace the broad domain allowance with narrower package allowances.
- [ ] Add explicit domain bans for `java.io..`, `java.net..`, `java.security..`, `java.util.jar..`, `java.util.zip..`, `org.slf4j..`, `ch.qos.logback..`, `org.yaml.snakeyaml..`, `org.springframework..`.
- [ ] Add explicit class bans for `java.nio.file.Files`, `java.nio.file.StandardCopyOption`, and `java.nio.file.FileAlreadyExistsException`.
- [ ] Keep `java.nio.file.Path` allowed.

#### Validation

- [ ] Command: `./mvnw -Dtest=HexagonalArchTest test`
- [ ] Expected result: fails and names the current domain offenders.

#### Acceptance Criteria

- [ ] The failure proves the rule catches technical details in domain.
- [ ] The failure is specific enough to guide remediation without banning accepted `Path` value usage.

### Milestone 2 - Move Child Process Command Construction To Secondary

#### Goal

Remove process command assembly from domain.

#### Changes

- [ ] Move `JavaProcessChildLauncher` behavior from `src/main/java/com/seed4j/cli/bootstrap/domain` to a new secondary adapter.
- [ ] Make `ChildProcessLauncher` and `JavaChildProcessRequest` accessible as domain port/request types if secondary needs them.
- [ ] Keep command ordering stable: sorted `-D` properties, `-cp`, executable JAR, main class, then CLI args.
- [ ] Update `Seed4JCliLauncherFactory` or composition so secondary adapter is injected, not constructed in domain.
- [ ] Move `JavaProcessChildLauncherTest` to secondary tests.

#### Validation

- [ ] Command: `./mvnw -Dtest=JavaProcessChildLauncherTest,Seed4JCliLauncherFactoryTest test`
- [ ] Expected result: command construction and factory/composition behavior pass.

#### Acceptance Criteria

- [ ] Domain no longer builds shell/JVM command lists.
- [ ] Standard child process launch still produces the same command.

### Milestone 3 - Move Packaged JAR Detection And Bootstrap Diagnostics To Ports

#### Goal

Remove `Files.isRegularFile` and Logback access from `Seed4JCliLauncher`.

#### Changes

- [ ] Add domain ports for packaged executable detection and bootstrap diagnostics.
- [ ] Implement packaged JAR detection in secondary using `Files.isRegularFile` and `.jar` suffix.
- [ ] Implement debug logging enablement in secondary using Logback APIs.
- [ ] Inject these ports from pre-Spring composition.
- [ ] Update `Seed4JCliLauncherTest` to use fakes for these ports.

#### Validation

- [ ] Command: `./mvnw -Dtest=Seed4JCliLauncherTest,HexagonalArchTest test`
- [ ] Expected result: launcher behavior remains green except remaining planned architecture violations.

#### Acceptance Criteria

- [ ] Standard mode outside packaged JAR still falls back locally and warns.
- [ ] Extension mode outside packaged JAR still fails before child process.
- [ ] Debug mode still enables bootstrap diagnostics for extension mode.

### Milestone 4 - Move Runtime Extension JAR Inspection To Secondary

#### Goal

Move manifest and nested library inspection out of domain while preserving library selection decisions.

#### Changes

- [ ] Move `RuntimeExtensionStartClassResolver` implementation to secondary behind a domain port.
- [ ] Move JAR/nested-JAR/`pom.properties` parsing from `RuntimeExtensionLoaderPathResolver` to secondary.
- [ ] Keep `RuntimeLibraryIdentity`, `RuntimeLibraryEntry`, `RuntimeLibraryIdentityResolution`, `RuntimeLibraryVersionComparator`, and missing-library selection as domain concepts.
- [ ] Remove direct logging from domain library decision types; expose diagnostics as return values or let secondary log around adapter decisions.
- [ ] Move technical resolver tests to secondary and keep pure library selection tests in domain.

#### Validation

- [ ] Command: `./mvnw -Dtest=RuntimeExtensionStartClassResolverTest,RuntimeExtensionLoaderPathResolverTest,RuntimeExtensionMissingLibrariesSelectorTest,HexagonalArchTest test`
- [ ] Expected result: technical tests pass in secondary, pure selector tests pass in domain, architecture failures shrink.

#### Acceptance Criteria

- [ ] Invalid/missing `Start-Class` errors remain unchanged.
- [ ] `loader.path` output remains byte-for-byte equivalent for covered cases.
- [ ] Library conflict behavior remains unchanged.

### Milestone 5 - Move Overlay Cache And Cache Identity To Secondary

#### Goal

Move filesystem cache materialization and SHA-256 file reading out of domain.

#### Changes

- [ ] Move `RuntimeExtensionOverlayCache` implementation to secondary behind a domain port.
- [ ] Move `RuntimeExtensionCacheIdentityResolver` file-reading SHA-256 behavior to secondary.
- [ ] Keep `RuntimeExtensionCacheIdentity` as a domain value.
- [ ] Keep path traversal, global resource filtering, staging cleanup, and cache reuse behavior unchanged.
- [ ] Move `RuntimeExtensionOverlayCacheTest` and `RuntimeExtensionCacheIdentityResolverTest` to secondary tests.

#### Validation

- [ ] Command: `./mvnw -Dtest=RuntimeExtensionOverlayCacheTest,RuntimeExtensionCacheIdentityResolverTest,Seed4JCliLauncherTest,HexagonalArchTest test`
- [ ] Expected result: overlay/cache behavior passes and architecture test no longer reports these classes in domain.

#### Acceptance Criteria

- [ ] Overlay classes are materialized under the same cache root.
- [ ] Existing cache reuse works.
- [ ] Staging cleanup still happens on invalid JAR and I/O failures.
- [ ] Path traversal entries are still rejected.

### Milestone 6 - Recompose Bootstrap And Run Focused Public Path Checks

#### Goal

Wire all ports and adapters through the pre-Spring composition root.

#### Changes

- [ ] Update `src/main/java/com/seed4j/cli/bootstrap/composition/PreSpringBootstrapConfiguration.java`.
- [ ] Update `src/main/java/com/seed4j/cli/bootstrap/domain/Seed4JCliLauncherFactory.java` or remove it if it becomes wiring-only.
- [ ] Keep `application` free of `..infrastructure..` dependencies.
- [ ] Keep `primary` free of direct `secondary` dependencies.

#### Validation

- [ ] Command: `./mvnw -Dtest=HexagonalArchTest,PreSpringBootstrapConfigurationTest,PreSpringBootstrapApplicationServiceTest,Seed4JCliAppTest test`
- [ ] Expected result: architecture and bootstrap public path tests pass.

#### Acceptance Criteria

- [ ] Pre-Spring bootstrap still creates a runnable launcher.
- [ ] Composition remains the only place that wires secondary adapters into the bootstrap flow.

### Milestone 7 - Broad Validation

#### Goal

Confirm behavior and formatting after all focused refactors.

#### Changes

- [ ] Run focused Maven tests for all moved/refactored packages.
- [ ] Run full unit test suite.
- [ ] Run formatting check.
- [ ] Record validation outcomes in this ExecPlan.

#### Validation

- [ ] Command: `./mvnw test`
- [ ] Expected result: all unit/component tests pass.
- [ ] Command: `npm run prettier:check`
- [ ] Expected result: all supported files are formatted.
- [ ] Ask user to run: `./mvnw clean verify`
- [ ] Expected result from user: exit code and concise failure summary if any.

#### Acceptance Criteria

- [ ] `HexagonalArchTest` passes.
- [ ] No direct technical dependencies remain in bootstrap domain beyond accepted value-level `Path`.
- [ ] Runtime extension tests pass.
- [ ] User receives clear final validation status.

## Progress

- [x] ExecPlan saved to the requested shared path
- [x] Milestone 1 started
- [x] Milestone 1 completed
- [x] Milestone 2 started
- [x] Milestone 2 completed
- [x] Milestone 3 started
- [x] Milestone 3 completed
- [x] Milestone 4 started
- [x] Milestone 4 completed
- [x] Milestone 5 started
- [x] Milestone 5 completed
- [x] Milestone 6 started
- [x] Milestone 6 completed
- [x] Milestone 7 started
- [x] Milestone 7 completed

## Decisions

- Decision: Use `tdd-strict-autonomous-quiet` for implementation.
  Rationale: The change is cross-cutting and architecture-sensitive; quiet strict TDD keeps cycles small without flooding routine progress.
  Date/Author: 2026-06-03 / Codex

- Decision: Keep `Path` allowed in domain for this refactor.
  Rationale: `Seed4JCliHome`, `RuntimeExtensionJarPath`, and repository ports already model paths as domain boundary values. Removing all `Path` usage would be a separate broader design change.
  Date/Author: 2026-06-03 / Codex

- Decision: First change is the architecture test RED.
  Rationale: The user explicitly wants the project to stop ignoring technical packages before applying corrections.
  Date/Author: 2026-06-03 / Codex

- Decision: Add a `BootstrapOutput` port even though System.err was initially out of scope.
  Rationale: After moving JAR/cache/logging concerns, direct `System.err.println` was the last architecture violation. Keeping it would make the hardened domain rule permanently fail, so secondary now owns stderr writes while the domain still chooses message intent.
  Date/Author: 2026-06-03 / Codex

## Risks and Mitigations

- Risk: Tightening `HexagonalArchTest` too broadly may flag acceptable domain value usage.
  Mitigation: Ban concrete technical packages/classes first and keep `Path` temporarily allowed.

- Risk: Moving package-private domain classes to secondary can force public API expansion.
  Mitigation: Promote only true ports/request DTOs to public/package-accessible types; avoid exposing implementation helpers.

- Risk: Behavior changes in extension launch can be subtle because `loader.path` and system properties are exact strings.
  Mitigation: Preserve existing tests and assert exact property values through public launcher behavior.

- Risk: Logging assertions may become brittle after removing logging from domain.
  Mitigation: Keep logging tests at secondary adapter boundaries or replace with explicit diagnostics behavior.

## Validation Strategy

1. Follow strict quiet TDD for every milestone.
2. For each cycle, add one failing test or architecture rule first.
3. Run the full relevant focused suite every cycle.
4. Run a public-path checkpoint at least every two cycles, usually through `Seed4JCliLauncherTest`, `PreSpringBootstrapConfigurationTest`, or packaged runtime extension tests.
5. Finish with `./mvnw test` and `npm run prettier:check`.
6. Ask the user to run `./mvnw clean verify`; do not run it automatically unless explicitly requested.

Validation completed on 2026-06-03:

- `./mvnw -Dtest=HexagonalArchTest test`: failed first with the expected domain technical offenders.
- `./mvnw -Dtest=JavaProcessChildLauncherTest,Seed4JCliLauncherFactoryTest test`: passed.
- `./mvnw -Dtest=Seed4JCliLauncherTest test`: passed.
- `./mvnw -Dtest=RuntimeExtensionStartClassResolverTest,RuntimeExtensionLoaderPathResolverTest,RuntimeExtensionMissingLibrariesSelectorTest,RuntimeExtensionOverlayCacheTest,RuntimeExtensionCacheIdentityResolverTest,Seed4JCliLauncherTest,HexagonalArchTest test`: passed.
- `./mvnw -Dtest=HexagonalArchTest,PreSpringBootstrapConfigurationTest,PreSpringBootstrapApplicationServiceTest,Seed4JCliAppTest test`: passed.
- `./mvnw -Dtest=ExtensionRuntimeBootstrapInProcessTest test`: passed after updating the missed in-process test wiring.
- `npm run prettier:check`: failed before formatting, then passed after `npm run prettier:format`.
- `./mvnw test`: passed after formatting, 491 tests with no failures or errors.

## Rollout and Recovery

This is an internal refactor with no intended CLI behavior change. Rollout is safe when focused tests, architecture tests, unit tests, and formatting checks pass.

Recovery is to revert the refactor commit if runtime launch behavior regresses. Because milestones are small, a partial rollback can also target the specific moved adapter while keeping the hardened architecture rule only after all violations are fixed.

## Lessons Learned

- Current architecture tests can pass even when domain performs JAR/filesystem work because `java..` and logging packages are broadly allowed.
- The first architecture fix must distinguish `Path` as a value from `Files`/JAR/process/logging as technical behavior.
- Runtime extension behavior is well covered, but many tests currently live in `bootstrap/domain` because technical implementation classes live there; moving tests with classes is part of the refactor, not incidental churn.
- The ExecPlan file named in the handoff was missing from this checkout, so it was recreated from the user-provided plan before implementation.
- The hardened architecture RED names the expected bootstrap domain offenders, including `RuntimeExtensionLoaderPathResolver`, `RuntimeExtensionOverlayCache`, `RuntimeExtensionCacheIdentityResolver`, `RuntimeExtensionStartClassResolver`, `Seed4JCliLauncher`, `RuntimeExtensionMissingLibrariesSelector`, and `RuntimeLibraryIdentityResolution`.
- Moving runtime extension technical classes to secondary required making domain value and decision types public where adapter implementations legitimately consume them.
- Once technical adapter classes moved out of domain, direct `System.err.println` became visible as the final domain technical dependency.
- In-process extension bootstrap tests were an important missed public path; they caught the legacy constructor's lack of real extension adapters after the refactor.
