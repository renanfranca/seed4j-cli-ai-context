# Refactor Runtime Extension Spring Wiring

This ExecPlan is a living document. Keep `Progress`, `Decisions`, `Risks and Mitigations`, and `Lessons Learned` up to date as work advances.

Safety boundary: This task is limited to authorized, defensive maintenance of this repository. Do not provide offensive guidance or policy-bypassing instructions.

## Purpose / Big Picture

The runtime extension install command should keep working exactly as before, but the Spring wiring should better match the repository architecture. Users still run `seed4j extension install ...` and receive installed runtime artifacts under `~/.config/seed4j-cli/runtime/active/`, while the code stops leaking raw `user.home` into the Spring application flow.

This matters because `RuntimeExtensionConfiguration` is the business concept for the active extension paths. Resolving it once at the Spring infrastructure boundary makes the application service and domain collaborators easier to reason about and test.

## Scope

In scope:

- Move Spring-only runtime extension bean wiring out of `src/main/java/com/seed4j/cli/bootstrap/composition`.
- Make `RuntimeExtensionApplicationService` a Spring `@Service`.
- Adapt `user.home` to `RuntimeExtensionConfiguration` inside `bootstrap.infrastructure.secondary`.
- Refactor the Spring application flow so `RuntimeExtensionInstaller` and `RuntimeExtensionModeEnabler` receive `RuntimeExtensionConfiguration`, not raw `Path userHome`.
- Update tests and fixtures affected by constructor changes.

Out of scope:

- Do not refactor the pre-Spring launcher path in `Seed4JCliLauncher`; it may continue using `RuntimeExtensionConfiguration.withDefaultPaths(userHome)`.
- Do not change CLI syntax, output text, runtime config file format, or external configuration keys.
- Do not change runtime artifact target paths.

## Definitions

`user.home` is the JVM system property that points to the current user's home directory.

`RuntimeExtensionConfiguration` is the domain record that holds the active runtime extension jar path and metadata path.

`Spring application flow` means the object graph created after Spring Boot starts `Seed4JCliApp`.

`Pre-Spring flow` means bootstrap code that runs before Spring Boot starts and decides whether the CLI should run in standard mode or extension mode.

`Secondary infrastructure` is the hexagonal architecture package area for adapters and framework integration that the domain/application code drives.

## Existing Context

`src/main/java/com/seed4j/cli/bootstrap/composition/RuntimeExtensionApplicationConfiguration.java` is currently annotated with `@Configuration`. It creates `RuntimeModeConfigurationRepository`, `RuntimeExtensionArtifactsRepository`, and `RuntimeExtensionApplicationService` beans. It injects `@Value("${user.home}")` and converts that string to `Path`.

`src/main/java/com/seed4j/cli/bootstrap/application/RuntimeExtensionApplicationService.java` is currently not a Spring stereotype. Its constructor receives `Path userHome`, `RuntimeModeConfigurationRepository`, and `RuntimeExtensionArtifactsRepository`, then creates a `RuntimeExtensionInstaller` internally.

`src/main/java/com/seed4j/cli/bootstrap/domain/RuntimeExtensionInstaller.java` currently stores `Path userHome` and derives `RuntimeExtensionConfiguration.withDefaultPaths(userHome)` during `install`.

`src/main/java/com/seed4j/cli/bootstrap/domain/RuntimeExtensionModeEnabler.java` also stores `Path userHome` and derives `RuntimeExtensionConfiguration.withDefaultPaths(userHome)` during `enable`.

`src/main/java/com/seed4j/cli/bootstrap/infrastructure/secondary/RuntimeSelectionConfiguration.java` shows an existing local pattern: package-private Spring `@Configuration` classes can live in `bootstrap.infrastructure.secondary`.

`src/main/java/com/seed4j/cli/bootstrap/composition/PreSpringBootstrapConfiguration.java` is not a Spring `@Configuration`; it composes the pre-Spring bootstrap object graph manually and should remain in `bootstrap.composition`.

## Desired End State

`bootstrap.composition.RuntimeExtensionApplicationConfiguration` no longer exists.

A new package-private Spring configuration class exists at `src/main/java/com/seed4j/cli/bootstrap/infrastructure/secondary/RuntimeExtensionSpringConfiguration.java`.

`RuntimeExtensionSpringConfiguration` creates these beans:

- `RuntimeExtensionConfiguration`, using `RuntimeExtensionConfiguration.withDefaultPaths(Path.of(userHomePath))`.
- `RuntimeModeConfigurationRepository`, using `new FileSystemRuntimeModeConfigurationRepository(userHome)`.
- `RuntimeExtensionArtifactsRepository`, using `new FileSystemRuntimeExtensionArtifactsRepository()`.
- `RuntimeExtensionInstaller`, wired with `RuntimeExtensionConfiguration`, `RuntimeModeConfigurationRepository`, and `RuntimeExtensionArtifactsRepository`.
- `RuntimeExtensionModeEnabler`, wired with `RuntimeExtensionConfiguration` and `RuntimeModeConfigurationRepository`.

`RuntimeExtensionApplicationService` is annotated with `@Service` and receives a `RuntimeExtensionInstaller` through its constructor.

Runtime extension install behavior remains unchanged: installing a valid extension jar writes the active jar, metadata, and config file under the same paths as before.

## Milestones

### Milestone 1 - Refactor Domain Constructors

#### Goal

Make the domain install and enable services receive the already resolved `RuntimeExtensionConfiguration`.

#### Changes

- [ ] Edit `src/main/java/com/seed4j/cli/bootstrap/domain/RuntimeExtensionInstaller.java` so its public constructor accepts `RuntimeExtensionConfiguration runtimeExtensionConfiguration` instead of `Path userHome`.
- [ ] Store `RuntimeExtensionConfiguration` as a field in `RuntimeExtensionInstaller`.
- [ ] Remove `RuntimeExtensionConfiguration.withDefaultPaths(userHome)` from `RuntimeExtensionInstaller.install`.
- [ ] Edit `src/main/java/com/seed4j/cli/bootstrap/domain/RuntimeExtensionModeEnabler.java` so its constructor accepts `RuntimeExtensionConfiguration runtimeExtensionConfiguration` instead of `Path userHome`.
- [ ] Remove `RuntimeExtensionConfiguration.withDefaultPaths(userHome)` from `RuntimeExtensionModeEnabler.enable`.
- [ ] Add `Assert.notNull` constructor checks for newly injected mandatory collaborators if missing.

#### Validation

- [ ] Command: `./mvnw -Dtest=RuntimeExtensionInstallerTest,RuntimeExtensionModeEnablerTest test`
- [ ] Expected result: both test classes pass after updating tests for the constructor change.

#### Acceptance Criteria

- [ ] `RuntimeExtensionInstaller` no longer imports or stores `java.nio.file.Path` solely for `userHome`.
- [ ] `RuntimeExtensionModeEnabler` no longer imports or stores `java.nio.file.Path` solely for `userHome`.
- [ ] Existing behavior remains covered by updated tests.

### Milestone 2 - Make Application Service a Spring Service

#### Goal

Align the application service with Seed4J's accepted convention of annotating application services with Spring stereotypes.

#### Changes

- [ ] Edit `src/main/java/com/seed4j/cli/bootstrap/application/RuntimeExtensionApplicationService.java`.
- [ ] Add `@Service`.
- [ ] Change the constructor to receive `RuntimeExtensionInstaller runtimeExtensionInstaller`.
- [ ] Store the injected installer directly.
- [ ] Add `Assert.notNull("runtimeExtensionInstaller", runtimeExtensionInstaller)` in the constructor.
- [ ] Keep `install(RuntimeExtensionInstallRequest request)` behavior unchanged.

#### Validation

- [ ] Command: `./mvnw -Dtest=RuntimeExtensionApplicationServiceTest test`
- [ ] Expected result: the application service tests pass after updating them to inject a `RuntimeExtensionInstaller`.

#### Acceptance Criteria

- [ ] `RuntimeExtensionApplicationService` no longer creates `RuntimeExtensionInstaller` itself.
- [ ] `RuntimeExtensionApplicationService` is discoverable by Spring component scanning.
- [ ] The application service still delegates install requests and validates null requests.

### Milestone 3 - Move Spring Wiring to Secondary Infrastructure

#### Goal

Replace the existing composition class with Spring infrastructure wiring that adapts `user.home` once.

#### Changes

- [ ] Delete `src/main/java/com/seed4j/cli/bootstrap/composition/RuntimeExtensionApplicationConfiguration.java`.
- [ ] Add `src/main/java/com/seed4j/cli/bootstrap/infrastructure/secondary/RuntimeExtensionSpringConfiguration.java`.
- [ ] Mark the new class `@Configuration` and keep it package-private.
- [ ] Add a `RuntimeExtensionConfiguration` bean using `@Value("${user.home}") String userHomePath`, `Assert.notBlank("userHomePath", userHomePath)`, and `Path.of(userHomePath)`.
- [ ] Add a `RuntimeModeConfigurationRepository` bean using `FileSystemRuntimeModeConfigurationRepository`.
- [ ] Add a `RuntimeExtensionArtifactsRepository` bean using `FileSystemRuntimeExtensionArtifactsRepository`.
- [ ] Add `RuntimeExtensionInstaller` and `RuntimeExtensionModeEnabler` beans wired from the domain ports and `RuntimeExtensionConfiguration`.
- [ ] Do not add a `RuntimeExtensionApplicationService` bean manually because `@Service` now provides it.

#### Validation

- [ ] Command: `./mvnw -Dtest=RuntimeExtensionSpringConfigurationTest test`
- [ ] Expected result: the new Spring configuration test proves the expected beans are created and scoped to the provided `user.home`.

#### Acceptance Criteria

- [ ] No production Spring runtime extension wiring remains in `bootstrap.composition`.
- [ ] `RuntimeExtensionSpringConfiguration` lives in `bootstrap.infrastructure.secondary`.
- [ ] The Spring context can create `RuntimeExtensionApplicationService` through component scanning and the new infrastructure beans.

### Milestone 4 - Update Tests and Fixtures

#### Goal

Keep the test suite aligned with the new constructor boundaries and prove unchanged user-visible install behavior.

#### Changes

- [ ] Delete or replace `src/test/java/com/seed4j/cli/bootstrap/composition/RuntimeExtensionApplicationConfigurationTest.java`.
- [ ] Add `src/test/java/com/seed4j/cli/bootstrap/infrastructure/secondary/RuntimeExtensionSpringConfigurationTest.java`.
- [ ] Update `src/test/java/com/seed4j/cli/bootstrap/application/RuntimeExtensionApplicationServiceTest.java`.
- [ ] Update `src/test/java/com/seed4j/cli/bootstrap/domain/RuntimeExtensionInstallerTest.java`.
- [ ] Update `src/test/java/com/seed4j/cli/bootstrap/domain/RuntimeExtensionModeEnablerTest.java`.
- [ ] Update primary command fixtures that manually construct `RuntimeExtensionApplicationService`, especially `src/test/java/com/seed4j/cli/command/infrastructure/primary/CliFixture.java` and `src/test/java/com/seed4j/cli/command/infrastructure/primary/ExtensionInstallCommandTest.java`.
- [ ] Preserve explicit Given/When/Then structure and inline assertions where practical.

#### Validation

- [ ] Command: `./mvnw -Dtest=RuntimeExtensionApplicationServiceTest,RuntimeExtensionInstallerTest,RuntimeExtensionModeEnablerTest,RuntimeExtensionSpringConfigurationTest,ExtensionInstallCommandTest test`
- [ ] Expected result: all focused tests pass.

#### Acceptance Criteria

- [ ] Tests construct `RuntimeExtensionConfiguration.withDefaultPaths(userHome)` explicitly when outside Spring.
- [ ] Spring wiring behavior is tested through `ApplicationContextRunner`.
- [ ] CLI install command tests continue proving the observable install behavior.

### Milestone 5 - Full Repository Validation

#### Goal

Confirm the refactor did not break broader runtime, formatting, coverage, or style checks.

#### Changes

- [ ] Run formatting check and inspect failures before applying any formatter.
- [ ] If formatting fails, run `npm run prettier:format`, then inspect the resulting diff.
- [ ] Run full Maven verification.

#### Validation

- [ ] Command: `npm run prettier:check`
- [ ] Expected result: formatting passes. If it fails only due to changed files, run `npm run prettier:format` and re-run the check.
- [ ] Command: `./mvnw clean verify`
- [ ] Expected result: unit tests, integration tests, JaCoCo, Checkstyle, and coverage gates pass.

#### Acceptance Criteria

- [ ] `npm run prettier:check` passes.
- [ ] `./mvnw clean verify` passes.
- [ ] `git status --short` shows only intentional source/test/plan changes and no unrelated tracked-file edits.

## Progress

- [x] ExecPlan created
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

## Decisions

- Decision: Use `@Service` on `RuntimeExtensionApplicationService`.
  Rationale: Seed4J original accepts Spring stereotypes in application services, and this removes unnecessary manual bean creation.
  Date/Author: 2026-06-02 / Codex and Renan

- Decision: Keep the `user.home` boundary limited to the Spring application flow.
  Rationale: This improves the Spring runtime extension wiring without expanding scope into the pre-Spring launcher.
  Date/Author: 2026-06-02 / Codex and Renan

- Decision: Keep `PreSpringBootstrapConfiguration` in `bootstrap.composition`.
  Rationale: It is manual pre-Spring composition, not a Spring `@Configuration`.
  Date/Author: 2026-06-02 / Codex

- Decision: Name the new Spring wiring class `RuntimeExtensionSpringConfiguration`.
  Rationale: The class configures Spring beans for runtime extension collaborators, not the runtime extension domain concept itself.
  Date/Author: 2026-06-02 / Codex

## Risks and Mitigations

- Risk: Two beans of the same type may appear after adding `@Service` while keeping the old configuration class.
  Mitigation: Delete `RuntimeExtensionApplicationConfiguration` in the same milestone that introduces the new Spring wiring.

- Risk: Constructor changes may break tests and fixtures outside the obvious domain classes.
  Mitigation: Search for `new RuntimeExtensionApplicationService`, `new RuntimeExtensionInstaller`, and `new RuntimeExtensionModeEnabler` and update every call site.

- Risk: Moving configuration package may accidentally stop Spring from scanning the new class.
  Mitigation: Place the class under `com.seed4j.cli.bootstrap.infrastructure.secondary`, which is beneath `Seed4JCliApp` scan base package.

- Risk: Refactor may change runtime artifact paths.
  Mitigation: Keep `RuntimeExtensionConfiguration.withDefaultPaths(userHome)` as the single path derivation and assert exact jar, metadata, and config paths in tests.

- Risk: Applying the formatter may rewrite unrelated files.
  Mitigation: Run `npm run prettier:check` first. Only run `npm run prettier:format` if needed, then inspect `git diff`.

## Validation Strategy

1. Run focused constructor and wiring tests:
   `./mvnw -Dtest=RuntimeExtensionApplicationServiceTest,RuntimeExtensionInstallerTest,RuntimeExtensionModeEnablerTest,RuntimeExtensionSpringConfigurationTest,ExtensionInstallCommandTest test`
2. Run formatting validation:
   `npm run prettier:check`
3. Run full repository validation:
   `./mvnw clean verify`
4. Confirm final diff:
   `git status --short`
   `git diff -- src/main/java src/test/java shared`

## Rollout and Recovery

This is an internal refactor with no intended CLI behavior change. Rollout is the normal release path after full validation passes.

If the change causes a regression, revert the commit containing this refactor. Recovery should restore the previous manual Spring bean configuration and constructors that accepted `Path userHome`.

No database migration, external configuration migration, or user action is required.

## Lessons Learned

- `bootstrap.composition` currently has two different meanings: pre-Spring manual composition and Spring runtime extension configuration. This refactor keeps only the pre-Spring composition there.
- `RuntimeSelectionConfiguration` already establishes that package-private Spring configuration classes can live in `bootstrap.infrastructure.secondary`.
- The selected boundary intentionally does not remove `user.home` from all domain code; `Seed4JCliLauncher` remains out of scope for this ExecPlan.
- After changing `RuntimeExtensionInstaller` to receive `RuntimeExtensionConfiguration`, the narrow domain test command could not compile until `RuntimeExtensionApplicationService` was updated too. Constructor refactors that cross production compilation units may require completing the adjacent milestone before validating.
- Focused validation passed with `./mvnw -Dtest=RuntimeExtensionApplicationServiceTest,RuntimeExtensionInstallerTest,RuntimeExtensionModeEnablerTest,RuntimeExtensionSpringConfigurationTest,ExtensionInstallCommandTest test`: 27 tests, no failures or errors.
- Formatter validation passed after running `npm run prettier:format` on touched files and re-running `npm run prettier:check`.
- Full validation passed with `./mvnw clean verify`: 483 unit tests, 6 integration tests, no failures or errors, and all coverage checks met.
