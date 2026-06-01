# Bind Runtime Selection Through Spring

This ExecPlan is a living document. Keep `Progress`, `Decisions`, `Risks`, and `Lessons Learned` up to date as work advances.

## Purpose / Big Picture

This refactor keeps JVM system properties as the parent-to-child process handoff protocol, but stops manually reconstructing runtime state from `System.getProperties()` inside the Spring child process. The child
process will consume `-Dseed4j.cli.runtime.*` through Spring Boot configuration binding and expose a `RuntimeSelection` bean. Users should observe the same CLI behavior, especially `seed4j --version`, while the
code becomes more aligned with Seed4J's infrastructure configuration style.

Safety boundary: This task is limited to authorized, defensive maintenance of this repository. Do not provide offensive guidance or policy-bypassing instructions.

## Scope

In scope: replace manual `RuntimeSelectionProvider` consumption with Spring-bound runtime selection; remove the manual `RuntimeSystemProperties` reader used by `--version`; keep `-D` system properties as the
process handoff format; update tests and formatting.

Out of scope: changing how the parent process decides standard versus extension mode; changing persisted `~/.config/seed4j-cli/config.yml`; changing runtime install/disable behavior; changing public CLI output
text except where tests already model version metadata precedence.

## Definitions

`Parent process` means the pre-Spring launcher flow that reads persisted runtime configuration and starts a child JVM when needed.

`Child process` means the Spring Boot CLI process launched with `-Dseed4j.cli.runtime.*` properties.

`RuntimeSelection` is the domain record describing the effective runtime mode plus optional extension distribution metadata.

`Handoff protocol` means the JVM system properties passed from parent to child, for example `-Dseed4j.cli.runtime.mode=extension`.

`Spring binding` means using Spring Boot `@ConfigurationProperties` or `@Value` to read values from the Spring `Environment`, which already includes JVM `-D` system properties.

## Existing Context

`Seed4JCliLauncher` already creates `JavaChildProcessRequest` with system properties such as `seed4j.cli.runtime.mode`, `seed4j.cli.runtime.distribution.id`, and `seed4j.cli.runtime.distribution.version`.

`Seed4JCommandsFactory` currently depends on `RuntimeSelectionProvider` and `RuntimeSystemProperties`. `CurrentProcessRuntimeSelectionProvider` delegates to `SystemPropertyRuntimeSelectionProvider`, which
manually reads a `Map<String, String>` copied from `System.getProperties()`.

`StandardRuntimeSystemProperties` manually exposes all JVM properties and is only used by `Seed4JCommandsFactory` to prefer `seed4j.cli.version` and `seed4j.cli.seed4j.version` in `--version` output.

The repository already uses `@Value` in primary adapters and the referenced Seed4J style places Spring configuration/properties classes in `infrastructure.secondary`.

## Desired End State

The parent process still passes runtime handoff data as JVM `-D` properties.

The Spring child process binds `seed4j.cli.runtime.*` into `RuntimeSelectionProperties`, then creates one `RuntimeSelection` bean from `bootstrap.infrastructure.secondary.RuntimeSelectionConfiguration`.

`Seed4JCommandsFactory` receives `RuntimeSelection` directly and receives version metadata through Spring-bound constructor values, not through manual `System.getProperties()` access.

The manual runtime selection provider classes and manual runtime system property reader are removed.

## Milestones

### Milestone 1 - Add Spring Runtime Selection Binding

#### Goal

Create the Spring infrastructure adapter that turns `seed4j.cli.runtime.*` properties into the domain `RuntimeSelection`.

#### Changes

- [ ] Add `src/main/java/com/seed4j/cli/bootstrap/infrastructure/secondary/RuntimeSelectionProperties.java` as a package-private `@ConfigurationProperties("seed4j.cli.runtime")` mutable properties class.
- [ ] Add `src/main/java/com/seed4j/cli/bootstrap/infrastructure/secondary/RuntimeSelectionConfiguration.java` as a package-private `@Configuration` with
      `@EnableConfigurationProperties(RuntimeSelectionProperties.class)`.
- [ ] Map missing `mode` to `RuntimeMode.STANDARD`.
- [ ] Map `distribution.id` to `Optional<RuntimeDistributionId>` and `distribution.version` to `Optional<RuntimeDistributionVersion>`.
- [ ] Treat blank optional distribution fields as absent after trimming.
- [ ] Do not inject `RuntimeSelection` into `RuntimeExtensionApplicationService`; it is not needed by install behavior.

#### Validation

- [ ] Command: `./mvnw -Dtest=RuntimeSelectionConfigurationTest test`
- [ ] Expected result: focused tests pass and prove default standard mode plus extension metadata binding.

#### Acceptance Criteria

- [ ] A Spring context with no `seed4j.cli.runtime.*` values exposes standard `RuntimeSelection`.
- [ ] A Spring context with extension runtime properties exposes `RuntimeMode.EXTENSION`, typed distribution id, and typed distribution version.
- [ ] The implementation does not use `System.getProperties()` to build `RuntimeSelection`.

### Milestone 2 - Wire Commands to RuntimeSelection Bean

#### Goal

Make command rendering consume the Spring-created `RuntimeSelection` bean directly.

#### Changes

- [ ] Update `src/main/java/com/seed4j/cli/command/infrastructure/primary/Seed4JCommandsFactory.java` to inject `RuntimeSelection` instead of `RuntimeSelectionProvider`.
- [ ] Replace `runtimeSelectionProvider.runtimeSelection()` with the injected `RuntimeSelection`.
- [ ] Remove `src/main/java/com/seed4j/cli/command/infrastructure/primary/RuntimeSelectionProvider.java`.
- [ ] Remove `src/main/java/com/seed4j/cli/command/infrastructure/primary/CurrentProcessRuntimeSelectionProvider.java`.
- [ ] Remove `src/main/java/com/seed4j/cli/command/infrastructure/primary/SystemPropertyRuntimeSelectionProvider.java`.
- [ ] Delete the matching provider tests and update `CliFixture` to pass `RuntimeSelection` directly.

#### Validation

- [ ] Command: `./mvnw -Dtest=Seed4JCommandsFactoryTest,RuntimeSelectionConfigurationTest test`
- [ ] Expected result: version output tests still pass with direct `RuntimeSelection` injection.

#### Acceptance Criteria

- [ ] `seed4j --version` still displays runtime mode, distribution id, and distribution version.
- [ ] No production code references `RuntimeSelectionProvider`.
- [ ] No production code manually parses `seed4j.cli.runtime.*` from a system property map.

### Milestone 3 - Remove Manual RuntimeSystemProperties Reader

#### Goal

Eliminate the remaining manual `System.getProperties()` reader used by version output.

#### Changes

- [ ] Update `Seed4JCommandsFactory` constructor to accept Spring-bound values for `seed4j.cli.version` and `seed4j.cli.seed4j.version`, using `@Value("${seed4j.cli.version:}")` and
      `@Value("${seed4j.cli.seed4j.version:}")`.
- [ ] Preserve the existing fallback order for CLI version: `seed4j.cli.version`, then `project.version`, then `unknown`.
- [ ] Preserve the existing fallback order for Seed4J version: `seed4j.cli.seed4j.version`, then `project.seed4j-version`, then resolved CLI version.
- [ ] Remove `src/main/java/com/seed4j/cli/command/infrastructure/primary/RuntimeSystemProperties.java`.
- [ ] Remove `src/main/java/com/seed4j/cli/command/infrastructure/primary/StandardRuntimeSystemProperties.java`.
- [ ] Update command tests and fixtures to pass explicit version strings rather than a fake system property map.

#### Validation

- [ ] Command: `./mvnw -Dtest=Seed4JCommandsFactoryTest test`
- [ ] Expected result: version precedence and safe fallback tests pass.

#### Acceptance Criteria

- [ ] `Seed4JCommandsFactory` no longer depends on a system-property reader abstraction.
- [ ] Existing `--version` behavior remains covered by tests.
- [ ] No production code calls `System.getProperties()` for runtime handoff state.

### Milestone 4 - Full Validation and Documentation Check

#### Goal

Verify the refactor does not change CLI behavior or formatting.

#### Changes

- [ ] Run targeted search to confirm removed symbols are gone.
- [ ] Run repository formatting check.
- [ ] Run full Maven verification.

#### Validation

- [ ] Command: `rg -n "RuntimeSelectionProvider|SystemPropertyRuntimeSelectionProvider|CurrentProcessRuntimeSelectionProvider|RuntimeSystemProperties|StandardRuntimeSystemProperties|System\\.getProperties\\(\\)"
src/main/java src/test/java`
- [ ] Expected result: no references to removed runtime selection/system-property reader classes; any remaining `System.getProperties()` reference must be unrelated and explicitly justified.
- [ ] Command: `npm run prettier:check`
- [ ] Expected result: formatting check passes.
- [ ] Command: `./mvnw clean verify`
- [ ] Expected result: unit tests, integration tests, Checkstyle, JaCoCo, and coverage gates pass.

#### Acceptance Criteria

- [ ] Full local validation passes.
- [ ] No documentation update is required because CLI flags, config files, and user-visible behavior are unchanged.

## Progress

- [x] ExecPlan saved to `shared/2026-06-01_REFACTOR_runtime-selection-spring-binding-exec-plan.md`
- [x] Milestone 1 started
- [ ] Milestone 1 completed
- [ ] Milestone 2 started
- [ ] Milestone 2 completed
- [ ] Milestone 3 started
- [ ] Milestone 3 completed
- [ ] Milestone 4 started
- [ ] Milestone 4 completed
- [ ] ExecPlan finalized with validation results and lessons learned

## Decisions

- Decision: Keep JVM `-Dseed4j.cli.runtime.*` system properties as the parent-to-child handoff protocol.
  Rationale: The data is execution-scoped, not persisted user configuration, and Spring Boot reads JVM system properties natively through the `Environment`.
  Date/Author: 2026-06-01 / Renan and Codex

- Decision: Consume runtime selection in the child through Spring Boot binding, not manual `System.getProperties()` parsing.
  Rationale: This aligns with Spring Boot conventions and with the Seed4J pattern of placing configuration/properties adapters in `infrastructure.secondary`.
  Date/Author: 2026-06-01 / Renan and Codex

- Decision: Place `RuntimeSelectionConfiguration` and `RuntimeSelectionProperties` in `com.seed4j.cli.bootstrap.infrastructure.secondary`.
  Rationale: They adapt external Spring configuration into a domain object; they are not application-service composition.
  Date/Author: 2026-06-01 / Renan and Codex

- Decision: Do not inject `RuntimeSelection` into `RuntimeExtensionApplicationService`.
  Rationale: The service installs runtime artifacts and does not need the current child process runtime selection.
  Date/Author: 2026-06-01 / Renan and Codex

- Decision: Remove both the manual runtime selection provider and the manual runtime system-property reader.
  Rationale: Cleaning both keeps one consistent rule: child process runtime handoff values are consumed through Spring, while `-D` remains only the transport.
  Date/Author: 2026-06-01 / Renan and Codex

## Risks and Mitigations

- Risk: Spring binding changes property precedence because `Environment` includes config files, env vars, command-line args, and JVM system properties.
  Mitigation: Record this as intentional; the parent still sends `-D` values, and Spring Boot's normal precedence applies.

- Risk: Removing provider interfaces may break test fixtures that instantiate command wiring manually.
  Mitigation: Update `CliFixture` and focused command tests in the same milestone as the constructor change.

- Risk: Version output fallback behavior may change while removing `RuntimeSystemProperties`.
  Mitigation: Keep existing `Seed4JCommandsFactoryTest` cases for dedicated runtime version values and safe fallback.

- Risk: `RuntimeSelectionProperties` may compile but fail to bind in a real Spring context.
  Mitigation: Use a Spring context-based test, not only direct unit instantiation.

## Validation Strategy

1. Run focused tests for the new runtime selection binding.
2. Run focused command tests for `--version` output and fixture wiring.
3. Run `rg` checks to verify removed manual readers are gone.
4. Run `npm run prettier:check`.
5. Run `./mvnw clean verify`.

## Rollout and Recovery

This is an internal refactor with unchanged CLI contract. Roll out by merging after full validation passes.

If runtime startup or `--version` behavior regresses, revert the commit that applies this ExecPlan. Because no persisted config format or CLI command syntax changes, recovery does not require data migration.

## Lessons Learned

The parent process can keep using JVM system properties as an effective ephemeral handoff channel without forcing the child process to manually read `System.getProperties()`. Spring Boot already treats `-D`
properties as configuration input, so the cleaner boundary is transport by JVM properties and consumption by Spring binding.
