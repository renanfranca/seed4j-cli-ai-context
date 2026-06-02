# Refatorar composição da extensão runtime

This ExecPlan is a living document. Keep `Progress`, `Decisions`, `Risks and Mitigations`, and `Lessons Learned` up to date as work advances.

Safety boundary: This task is limited to authorized, defensive maintenance of this repository. Do not provide offensive guidance or policy-bypassing instructions.

## Purpose / Big Picture

Refactor the runtime extension bootstrap composition so `RuntimeExtensionApplicationService` follows the Seed4J application-service flavor: the application layer receives domain values and ports, then constructs the pure domain service it orchestrates. The user-visible behavior must stay the same: `seed4j extension install` still installs the extension jar, writes metadata, updates `~/.config/seed4j-cli/config.yml`, and prints the same success/error messages. This matters because runtime extension installation currently works, but its composition makes the application service a thin wrapper around an already-mounted domain service and keeps the `user.home` path concept too implicit.

## Scope

In scope: introduce a dedicated domain type for the CLI home path, use it to derive runtime/config paths, refactor Spring and pre-Spring composition to use that type, and make `RuntimeExtensionApplicationService` construct `RuntimeExtensionInstaller`.

Out of scope: changing CLI command syntax or output, changing the `~/.config/seed4j-cli` filesystem layout, changing runtime extension metadata format, changing packaged bootstrap behavior beyond preserving it, and refactoring unrelated module/project/account/sample packages.

The current untracked `_temporary/` directory is unrelated and must not be modified or removed.

## Definitions

`Application service` means a Spring-managed use-case orchestrator in `src/main/java/com/seed4j/cli/bootstrap/application`.

`Domain service` means a framework-free class in `src/main/java/com/seed4j/cli/bootstrap/domain` that contains domain behavior, such as `RuntimeExtensionInstaller`.

`Port` means a domain interface implemented by infrastructure, such as `RuntimeModeConfigurationRepository` and `RuntimeExtensionArtifactsRepository`.

`Seed4JCliHome` is the new domain value object planned here. It wraps the user home directory path and exposes named paths used by the CLI runtime, such as the config file, active runtime extension jar, active runtime metadata, and runtime cache directory.

`Pre-Spring bootstrap` means code that runs before Spring Boot starts, mainly under `bootstrap/composition`, `bootstrap/infrastructure/primary`, and `bootstrap/infrastructure/secondary`.

## Existing Context

`RuntimeExtensionApplicationService` currently receives `RuntimeExtensionInstaller` directly and delegates `install(request)` to it. This makes the application service a wrapper around a fully constructed domain service.

`RuntimeExtensionSpringConfiguration` currently reads `@Value("${user.home}")`, creates `RuntimeExtensionConfiguration`, creates `FileSystemRuntimeModeConfigurationRepository`, creates `FileSystemRuntimeExtensionArtifactsRepository`, and also exposes a `RuntimeExtensionInstaller` bean.

`PreSpringBootstrapConfiguration` has no Spring container. It reads `PreSpringRuntimeEnvironment`, extracts `runtimeEnvironment.userHomePath()`, and manually wires `FileSystemRuntimeModeConfigurationRepository`, `SpringBootLocalCliRunner`, and `Seed4JCliLauncherFactory`.

`CurrentProcessPreSpringRuntimeEnvironmentReader` is the correct infrastructure adapter for reading `System.getProperty("user.home")` before Spring exists.

The Seed4J sample app supports this direction. `BeersApplicationService` receives the domain port `BeersRepository` and constructs pure domain services `BeersCreator` and `BeersRemover`. `Seed4jsampleAuthorizations` receives domain `RolesAccesses` and translates Spring Security data into domain types. This shows that application receiving domain values is acceptable; the concern is avoiding infrastructure implementation and already-mounted domain service injection.

## Desired End State

`RuntimeExtensionApplicationService` receives `RuntimeExtensionConfiguration`, `RuntimeModeConfigurationRepository`, and `RuntimeExtensionArtifactsRepository`, validates them, and constructs `RuntimeExtensionInstaller` internally.

`RuntimeExtensionInstaller` remains a pure domain class and is no longer exposed as a Spring bean.

`Seed4JCliHome` centralizes all default CLI runtime paths derived from the user home. Production code no longer derives `.config/seed4j-cli/...` paths from raw `Path userHome` except inside `Seed4JCliHome` and the adapter that reads the current process environment.

Spring composition reads `user.home` once, creates `Seed4JCliHome`, and derives domain/runtime beans from it.

Pre-Spring composition reads `user.home` through `CurrentProcessPreSpringRuntimeEnvironmentReader`, stores it as `Seed4JCliHome`, and passes that typed concept through launcher/configuration code.

All existing runtime extension install and bootstrap tests still pass, and new tests prove `Seed4JCliHome` path derivation.

## Milestones

### Milestone 1 - Introduce Seed4JCliHome

#### Goal

Create the domain type that names the CLI home concept and owns default path derivation.

#### Changes

Edit `src/main/java/com/seed4j/cli/bootstrap/domain/Seed4JCliHome.java` to add a new public record wrapping `Path path`.

Use `Assert.notNull("path", path)` in the compact constructor.

Expose `configPath()`, `runtimeExtensionConfiguration()`, and `runtimeCacheDirectory()`.

Make `runtimeExtensionConfiguration()` return a `RuntimeExtensionConfiguration` with the existing paths `.config/seed4j-cli/runtime/active/extension.jar` and `.config/seed4j-cli/runtime/active/metadata.yml`.

Edit `src/main/java/com/seed4j/cli/bootstrap/domain/RuntimeExtensionConfiguration.java` to remove `withDefaultPaths(Path userHome)` after all callers are migrated.

Add `src/test/java/com/seed4j/cli/bootstrap/domain/Seed4JCliHomeTest.java` with explicit assertions for config path, active extension jar path, active metadata path, runtime cache path, and null validation.

#### Validation

Command: `./mvnw -Dtest=Seed4JCliHomeTest test`

Expected result: Maven exits with code `0`, and the new unit test proves all paths still resolve under the provided home directory.

#### Acceptance Criteria

`Seed4JCliHome` is the only production type that knows the default `.config/seed4j-cli` path fragments.

`RuntimeExtensionConfiguration` remains a simple domain record containing explicit jar and metadata paths.

### Milestone 2 - Refactor Spring runtime extension composition

#### Goal

Make the Spring-managed runtime extension install flow match the Seed4J application-service flavor.

#### Changes

Edit `src/main/java/com/seed4j/cli/bootstrap/application/RuntimeExtensionApplicationService.java` so its constructor receives `RuntimeExtensionConfiguration`, `RuntimeModeConfigurationRepository`, and `RuntimeExtensionArtifactsRepository`.

Inside that constructor, validate each dependency with `Assert.notNull` and instantiate `new RuntimeExtensionInstaller(...)`.

Keep `install(RuntimeExtensionInstallRequest request)` behavior unchanged.

Edit `src/main/java/com/seed4j/cli/bootstrap/infrastructure/secondary/RuntimeExtensionSpringConfiguration.java` to expose a `Seed4JCliHome` bean from `@Value("${user.home}")`.

Change `runtimeExtensionConfiguration(...)` to derive configuration from `Seed4JCliHome`.

Change `runtimeModeConfigurationRepository(...)` to construct `FileSystemRuntimeModeConfigurationRepository` from `Seed4JCliHome`.

Remove the `RuntimeExtensionInstaller` bean method.

Keep `RuntimeExtensionArtifactsRepository` and `RuntimeExtensionModeEnabler` beans.

Update `src/test/java/com/seed4j/cli/command/infrastructure/primary/ExtensionInstallCommandTest.java` and `src/test/java/com/seed4j/cli/command/infrastructure/primary/CliFixture.java` so test fixtures construct `RuntimeExtensionApplicationService` from the same dependencies the application service now expects, not from `RuntimeExtensionInstaller`.

Keep `src/test/java/com/seed4j/cli/command/infrastructure/primary/ExtensionInstallSpringContextTest.java` focused on observable install behavior through the Spring command graph. Do not add assertions for bean counts or constructor wiring.

#### Validation

Command: `./mvnw -Dtest=ExtensionInstallCommandTest,ExtensionInstallSpringContextTest test`

Expected result: Maven exits with code `0`, extension install command still persists config/runtime artifacts, and command output still contains the same success paths and validation hints.

#### Acceptance Criteria

No Spring bean directly exposes `RuntimeExtensionInstaller`.

`RuntimeExtensionApplicationService` constructs `RuntimeExtensionInstaller`.

The Spring-context command test still installs an extension using `user.home` from Spring property binding.

### Milestone 3 - Propagate Seed4JCliHome through pre-Spring bootstrap

#### Goal

Remove primitive `Path userHome` composition from bootstrap runtime code and preserve pre-Spring behavior.

#### Changes

Edit `src/main/java/com/seed4j/cli/bootstrap/domain/PreSpringRuntimeEnvironment.java` so it stores `Seed4JCliHome cliHome` instead of `Path userHomePath`.

Edit `src/main/java/com/seed4j/cli/bootstrap/infrastructure/secondary/CurrentProcessPreSpringRuntimeEnvironmentReader.java` to create `new Seed4JCliHome(Path.of(System.getProperty("user.home")))`.

Edit `src/main/java/com/seed4j/cli/bootstrap/composition/PreSpringBootstrapConfiguration.java` to use `runtimeEnvironment.cliHome()` when constructing `FileSystemRuntimeModeConfigurationRepository`, `SpringBootLocalCliRunner`, and `Seed4JCliLauncherFactory` dependencies.

Edit `src/main/java/com/seed4j/cli/bootstrap/infrastructure/secondary/SpringBootLocalCliRunner.java` to receive `Seed4JCliHome` and use `cliHome.configPath()` instead of resolving `.config/seed4j-cli/config.yml` locally.

Edit `src/main/java/com/seed4j/cli/bootstrap/infrastructure/secondary/FileSystemRuntimeModeConfigurationRepository.java` to receive `Seed4JCliHome` and use `cliHome.configPath()`.

Edit `src/main/java/com/seed4j/cli/bootstrap/infrastructure/secondary/RuntimeModeConfigReader.java` so its public package-level methods receive a concrete config path instead of a user home path. Preserve existing user-facing error messages mentioning `~/.config/seed4j-cli/config.yml`.

Edit `src/main/java/com/seed4j/cli/bootstrap/domain/Seed4JCliLauncherFactory.java` and `src/main/java/com/seed4j/cli/bootstrap/domain/Seed4JCliLauncher.java` to pass and store `Seed4JCliHome`.

Edit `src/main/java/com/seed4j/cli/bootstrap/domain/RuntimeExtensionOverlayCache.java` to receive `Seed4JCliHome` and use `cliHome.runtimeCacheDirectory()`.

Update all affected tests that currently construct `PreSpringRuntimeEnvironment`, `FileSystemRuntimeModeConfigurationRepository`, `SpringBootLocalCliRunner`, or `Seed4JCliLauncher` with raw user home paths.

#### Validation

Command: `./mvnw -Dtest=CurrentProcessPreSpringRuntimeEnvironmentReaderTest,SpringBootLocalCliRunnerTest,FileSystemRuntimeModeConfigurationRepositoryTest,PreSpringBootstrapConfigurationTest,PreSpringBootstrapApplicationServiceTest,Seed4JCliLauncherFactoryTest,Seed4JCliLauncherTest test`

Expected result: Maven exits with code `0`, pre-Spring environment reading still uses the current process `user.home`, config reading still defaults to standard mode when config is missing, and launcher behavior remains unchanged.

#### Acceptance Criteria

Production code no longer passes a raw `Path userHome` through bootstrap composition when the concept is actually CLI home.

Path derivation for config, active extension runtime, and cache is centralized in `Seed4JCliHome`.

Pre-Spring bootstrap still works without Spring `@Value`.

### Milestone 4 - Run behavior and integration validation

#### Goal

Prove that the refactor preserved real runtime extension behavior.

#### Changes

No new production changes unless validation exposes a regression.

Update this ExecPlan `Progress`, `Decisions`, `Risks and Mitigations`, and `Lessons Learned` before and after each validation run.

#### Validation

Command: `./mvnw -Dtest=RuntimeExtensionInstallerTest,RuntimeExtensionModeEnablerTest,RuntimeExtensionModeDisablerTest,RuntimeSelectionTest,ExtensionInstallCommandTest,ExtensionInstallSpringContextTest test`

Expected result: Maven exits with code `0`, install, enable, disable, and runtime selection behavior remain unchanged.

Command: `./mvnw -Dit.test=ExtensionRuntimeBootstrapPackagedJarIT,ExtensionRuntimeBootstrapListPackagedJarIT,ExtensionRuntimeBootstrapApplyPackagedJarIT failsafe:integration-test failsafe:verify`

Expected result: Maven exits with code `0`, packaged runtime extension bootstrap still loads/list/applies extension behavior from a temporary user home.

Command: `npm run prettier:check`

Expected result: formatting check exits with code `0`.

Command: `./mvnw test`

Expected result: unit test suite exits with code `0`.

#### Acceptance Criteria

All targeted runtime extension and bootstrap tests pass.

No command output or persisted runtime file path changes.

No test is added solely to assert Spring wiring, bean counts, or collaborator call order.

## Progress

- [x] ExecPlan saved to `shared/2026-06-02_REFACTOR_runtime-extension-application-composition-exec-plan.md`
- [x] Milestone 1 started
- [x] Milestone 1 completed
- [x] Milestone 2 started
- [x] Milestone 2 completed
- [x] Milestone 3 started
- [x] Milestone 3 completed
- [x] Milestone 4 started
- [x] Milestone 4 completed
- [x] Final validation summary recorded

## Decisions

Decision: Preserve `~/.config/seed4j-cli/config.yml` path text in configuration error messages outside `Seed4JCliHome`.
Rationale: Those strings are user-facing diagnostics, not production path derivation logic, and changing them would violate the no-output-change scope.
Date/Author: 2026-06-02 / Codex

Decision: Introduce `Seed4JCliHome` as a domain value object rather than passing raw `Path userHome`.
Rationale: The path is not just a filesystem primitive; it defines where the CLI runtime config, active extension, metadata, and cache live.
Date/Author: 2026-06-02 / Codex with user direction

Decision: Allow application services to receive domain values and ports.
Rationale: Seed4J examples show `application` receiving domain ports and domain values; the architectural issue is injecting already-mounted domain services or concrete infrastructure, not touching domain types.
Date/Author: 2026-06-02 / Codex with user direction

Decision: Make `RuntimeExtensionApplicationService` construct `RuntimeExtensionInstaller`.
Rationale: This matches `BeersApplicationService`, which receives a repository port and constructs pure domain services.
Date/Author: 2026-06-02 / Codex with user direction

Decision: Keep `user.home` reads in adapters/composition roots only.
Rationale: Spring uses `@Value`; pre-Spring uses `CurrentProcessPreSpringRuntimeEnvironmentReader`. The application and domain should receive typed values, not read global process state.
Date/Author: 2026-06-02 / Codex with user direction

## Risks and Mitigations

Risk: Refactoring `PreSpringRuntimeEnvironment` and `Seed4JCliLauncher` may break packaged bootstrap behavior.
Mitigation: Run focused launcher tests and packaged Failsafe integration tests after the refactor.

Risk: Removing `RuntimeExtensionConfiguration.withDefaultPaths(Path)` may require many test updates.
Mitigation: Update tests to use `new Seed4JCliHome(path).runtimeExtensionConfiguration()` and keep assertions explicit.

Risk: `RuntimeModeConfigReader` error messages could accidentally change.
Mitigation: Preserve existing message text and run tests that assert invalid config messages.

Risk: The refactor could drift into implementation-detail tests.
Mitigation: Prefer behavior tests through command execution, repository behavior, path value object behavior, and packaged bootstrap integration.

Risk: The untracked `_temporary/` directory could be accidentally modified.
Mitigation: Ignore it entirely unless the user explicitly assigns work involving that directory.

## Validation Strategy

Run validation from narrow to broad.

First run `./mvnw -Dtest=Seed4JCliHomeTest test`.

Then run command and Spring install flow tests with `./mvnw -Dtest=ExtensionInstallCommandTest,ExtensionInstallSpringContextTest test`.

Then run bootstrap and launcher tests with `./mvnw -Dtest=CurrentProcessPreSpringRuntimeEnvironmentReaderTest,SpringBootLocalCliRunnerTest,FileSystemRuntimeModeConfigurationRepositoryTest,PreSpringBootstrapConfigurationTest,PreSpringBootstrapApplicationServiceTest,Seed4JCliLauncherFactoryTest,Seed4JCliLauncherTest test`.

Then run runtime extension domain tests with `./mvnw -Dtest=RuntimeExtensionInstallerTest,RuntimeExtensionModeEnablerTest,RuntimeExtensionModeDisablerTest,RuntimeSelectionTest test`.

Then run packaged integration tests with `./mvnw -Dit.test=ExtensionRuntimeBootstrapPackagedJarIT,ExtensionRuntimeBootstrapListPackagedJarIT,ExtensionRuntimeBootstrapApplyPackagedJarIT failsafe:integration-test failsafe:verify`.

Then run `npm run prettier:check`.

Then run `./mvnw test`.

Do not run `./mvnw clean verify` automatically. Ask the user to run it locally when the final gate is needed and provide the exit code plus concise failure summary if it fails.

## Rollout and Recovery

This is an internal refactor with no intended CLI behavior change. Rollout is normal merge after tests pass.

Recovery is straightforward: revert the refactor commit if packaged bootstrap or extension install behavior regresses.

If a regression is discovered before merge, restore the previous constructor wiring for the affected path, keep `Seed4JCliHome` only if it is already proven safe, and rerun the targeted tests before continuing.

## Lessons Learned

`./mvnw -Dtest=Seed4JCliHomeTest test` passed on 2026-06-02 with 2 tests, confirming config, active extension jar, active metadata, runtime cache path derivation, and null validation.

`./mvnw -Dtest=ExtensionInstallCommandTest,ExtensionInstallSpringContextTest test` passed on 2026-06-02 with 7 tests, confirming command-visible install behavior and Spring property based `user.home` composition.

`./mvnw -Dtest=CurrentProcessPreSpringRuntimeEnvironmentReaderTest,SpringBootLocalCliRunnerTest,FileSystemRuntimeModeConfigurationRepositoryTest,PreSpringBootstrapConfigurationTest,PreSpringBootstrapApplicationServiceTest,Seed4JCliLauncherFactoryTest,Seed4JCliLauncherTest test` passed on 2026-06-02 with 53 tests, confirming pre-Spring composition, config reading, local runner behavior, and launcher behavior.

`./mvnw -Dtest=RuntimeExtensionInstallerTest,RuntimeExtensionModeEnablerTest,RuntimeExtensionModeDisablerTest,RuntimeSelectionTest,ExtensionInstallCommandTest,ExtensionInstallSpringContextTest test` passed on 2026-06-02 with 52 tests, confirming runtime install, enable, disable, selection, and command behavior.

`./mvnw -Dit.test=ExtensionRuntimeBootstrapPackagedJarIT,ExtensionRuntimeBootstrapListPackagedJarIT,ExtensionRuntimeBootstrapApplyPackagedJarIT failsafe:integration-test failsafe:verify` passed on 2026-06-02 with 6 packaged integration tests.

`npm run prettier:check` initially reported formatting issues in 4 changed files; after formatting those files with Prettier, `npm run prettier:check` passed.

`./mvnw test` passed on 2026-06-02 with 481 tests.

Production search for `.config/seed4j-cli` now shows default path derivation only in `Seed4JCliHome`; remaining occurrences are preserved user-facing error messages in runtime config/mode classes.

The Seed4J sample app shows that application receiving domain values is acceptable; the key boundary is preventing application from depending on concrete infrastructure or process-global environment reads.

`RuntimeExtensionConfiguration` is domain, not a port. Treating it honestly avoids a false architecture rule.

Pre-Spring bootstrap is not an architecture exception; it is a manual composition root replacing Spring configuration before the Spring container exists.

The phrase “remove user.home” should mean “stop spreading raw `user.home` as an unnamed primitive,” not “pretend the CLI does not depend on the user home directory.”
