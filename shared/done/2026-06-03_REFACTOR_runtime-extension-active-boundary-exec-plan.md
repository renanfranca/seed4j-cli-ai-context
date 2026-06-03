# Refatorar Boundary do Runtime Extension Ativo

This ExecPlan is a living document. Keep `Progress`, `Decisions`, `Risks and Mitigations`, and `Lessons Learned` up to date as work advances.

## Purpose / Big Picture

This refactor corrects the hexagonal boundary of the active runtime extension flow. Users should observe the same behavior in `seed4j extension install`, `seed4j extension enable`, bootstrap in `extension` mode, and `seed4j --version`, while the domain stops knowing the physical layout of `~/.config/seed4j-cli/runtime/active/extension.jar` and `metadata.yml`.

The change matters because the current domain resolves files, parses YAML, validates JAR layout, and coordinates `IOException` in part of the runtime flow. That mixes business orchestration with filesystem details and makes the domain/application/secondary boundaries less reliable.

Safety boundary: This task is limited to authorized, defensive maintenance of this repository. Do not provide offensive guidance or policy-bypassing instructions.

## Scope

In scope:

- Remove `RuntimeExtensionConfiguration` and `RuntimeConfiguration` from the domain.
- Make runtime extension install, enable, and active selection depend on domain ports instead of concrete active artifact paths.
- Move active artifact filesystem layout, YAML parsing/writing, and JAR package validation to `bootstrap.infrastructure.secondary`.
- Preserve the current physical layout under `~/.config/seed4j-cli/runtime/active`.
- Preserve observable command output and behavior.
- Update affected domain, secondary, Spring, and primary tests.

Out of scope:

- Moving `Seed4JCliLauncher`, `RuntimeExtensionOverlayCache`, `RuntimeExtensionLoaderPathResolver`, `RuntimeExtensionStartClassResolver`, or nested JAR reading entirely to infrastructure.
- Changing the parent/child handoff protocol through `-Dseed4j.cli.runtime.*`.
- Changing user documentation unless a user-facing contract changes.
- Changing persisted file layout.

## Definitions

`Runtime extension ativo` is the installed extension used by the CLI when `seed4j.runtime.mode` is configured as `extension`.

`Artefatos ativos` are the current operational active runtime files: `extension.jar` and `metadata.yml`.

`Metadata de runtime` is the YAML containing `distribution.id` and `distribution.version`.

`Port` is a domain interface describing a capability without selecting a concrete technology.

`Adapter secondary` is an implementation of a domain port that talks to external mechanisms such as filesystem, YAML, or JDK JAR APIs.

`RuntimeSelection` is the domain value describing the effective runtime mode and optional extension distribution/JAR data.

`Seed4JCliHome` remains a documented domain concept for paths derived from `user.home`, but it must not expose active runtime extension configuration.

## Existing Context

`RuntimeSelection.resolve(...)` currently reads filesystem state, validates a JAR, and reads YAML metadata. `RuntimeExtensionInstaller`, `RuntimeExtensionModeEnabler`, and `Seed4JCliLauncher` currently use `RuntimeExtensionConfiguration` or `RuntimeConfiguration`, which expose active artifact paths to the domain.

`FileSystemRuntimeExtensionArtifactsRepository` already performs part of artifact installation, but its port contract receives `RuntimeExtensionConfiguration` from the domain. `FileSystemRuntimeModeConfigurationRepository` returns a `RuntimeModeChangePlan` whose `apply()` leaks checked `IOException` into domain signatures.

## Desired End State

The domain does not contain `RuntimeExtensionConfiguration` or `RuntimeConfiguration`. `RuntimeSelection` is pure and uses `Optional<RuntimeExtensionJarPath>` for the extension JAR path.

Active runtime resolution lives in a secondary adapter. JAR layout validation lives behind a domain port implemented by secondary. `RuntimeExtensionInstaller`, `RuntimeExtensionModeEnabler`, and `Seed4JCliLauncher` orchestrate behavior through ports without receiving the active artifact physical layout.

`RuntimeModeChangePlan.apply()` no longer declares `throws IOException`; secondary adapters convert technical failures to `InvalidRuntimeConfigurationException`.

## Milestones

### Milestone 1 - Make `RuntimeSelection` Pure

Change `RuntimeSelection` to factories only, remove `RuntimeConfiguration`, update child-process binding and tests, and remove filesystem scenarios from domain selection tests.

Validation:

- `./mvnw -Dtest=RuntimeSelectionConfigurationTest,Seed4JCommandsFactoryTest,JavaProcessChildLauncherTest test`
- `rg -n "RuntimeSelection\\.resolve|new RuntimeConfiguration|RuntimeConfiguration" src/main/java src/test/java`

### Milestone 2 - Introduce Active Runtime Ports

Update `RuntimeExtensionArtifactsRepository`, add installation/selection/package-validator/metadata domain contracts, and update `RuntimeExtensionInstaller` to depend on ports.

Validation:

- `./mvnw -Dtest=RuntimeExtensionInstallerTest test`

### Milestone 3 - Move Active Artifact Filesystem and Metadata to Secondary

Make `FileSystemRuntimeExtensionArtifactsRepository` own the active layout, add `FileSystemRuntimeExtensionSelectionRepository`, move YAML metadata parsing to secondary, and add `JarRuntimeExtensionPackageValidator`.

Validation:

- `./mvnw -Dtest=FileSystemRuntimeExtensionArtifactsRepositoryTest,FileSystemRuntimeExtensionSelectionRepositoryTest test`
- `rg -n "Files\\.|JarFile|Yaml|RuntimeMetadata" src/main/java/com/seed4j/cli/bootstrap/domain`

### Milestone 4 - Update Enable and Launcher

Make `RuntimeExtensionModeEnabler` and `Seed4JCliLauncher` use `RuntimeExtensionSelectionRepository`.

Validation:

- `./mvnw -Dtest=RuntimeExtensionModeEnablerTest,Seed4JCliLauncherTest test`
- `rg -n "runtimeExtensionConfiguration\\(|RuntimeExtensionConfiguration" src/main/java src/test/java`

### Milestone 5 - Adjust Spring Wiring and Command Path

Wire the new secondary adapters in Spring and update `RuntimeExtensionApplicationService` and command tests.

Validation:

- `./mvnw -Dtest=ExtensionInstallCommandTest,ExtensionInstallSpringContextTest test`

### Milestone 6 - Remove `IOException` Leak from Mode Change Plan

Change `RuntimeModeChangePlan.apply()` to no checked exception and map filesystem failures in secondary.

Validation:

- `./mvnw -Dtest=RuntimeExtensionInstallerTest,RuntimeExtensionModeEnablerTest,RuntimeExtensionModeDisablerTest,FileSystemRuntimeModeConfigurationRepositoryTest test`
- `rg -n "apply\\(\\) throws IOException|catch \\(IOException" src/main/java/com/seed4j/cli/bootstrap/domain`

### Milestone 7 - Final Cleanup and Broad Validation

Delete dead types/tests, confirm documentation does not need changes, run formatter check and JUnit suite.

Validation:

- `rg -n "RuntimeExtensionConfiguration|RuntimeConfiguration|RuntimeSelection\\.resolve|runtimeExtensionConfiguration\\(" src/main/java src/test/java`
- `npm run prettier:check`
- `./mvnw test`
- Ask the user to run `./mvnw clean verify` unless they explicitly request the agent to run it.

## Progress

- [x] ExecPlan saved to `shared/2026-06-03_REFACTOR_runtime-extension-active-boundary-exec-plan.md`
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
- [x] ExecPlan finalized with validation results and lessons learned

## Decisions

- Decision: Correct the full active runtime flow, not only `RuntimeExtensionConfiguration`.
  Rationale: The record is a symptom; the leak also includes `RuntimeSelection.resolve(...)`, YAML/JAR inspection, and mode-change `IOException`.
  Date/Author: 2026-06-03 / Renan and Codex

- Decision: Keep `Seed4JCliHome` in the domain for now.
  Rationale: The repository guidelines define it as the domain concept for paths derived from `user.home`; moving it would expand scope.
  Date/Author: 2026-06-03 / Renan and Codex

- Decision: Preserve physical layout and CLI output.
  Rationale: This is an architecture refactor; changing user contracts would add unnecessary risk.
  Date/Author: 2026-06-03 / Renan and Codex

- Decision: Move old `RuntimeSelection.resolve(...)` filesystem/YAML/JAR scenarios to `FileSystemRuntimeExtensionSelectionRepositoryTest`.
  Rationale: Keeping those tests in `bootstrap.domain` would preserve the old boundary through test design even after production code moved to secondary.
  Date/Author: 2026-06-03 / Codex

- Decision: Do not preserve compatibility overloads for old constructors or repositories in production.
  Rationale: Backward-compatible overloads would keep old active artifact configuration concepts reachable and make the boundary cleanup incomplete.
  Date/Author: 2026-06-03 / Codex

- Decision: Do not edit `README.md` or `documentation/Commands.md`.
  Rationale: CLI behavior, command output, config keys, and persisted layout did not change; this was an internal architecture refactor.
  Date/Author: 2026-06-03 / Codex

## Risks and Mitigations

- Risk: Command output or error messages change accidentally.
  Mitigation: Preserve current messages in secondary adapters and validate command/secondary tests.

- Risk: The refactor creates wide test churn.
  Mitigation: Move behavior tests from domain filesystem tests to secondary tests and keep domain tests fake-based.

- Risk: Spring wiring misses a new port.
  Mitigation: Run Spring context and command tests before broad validation.

## Validation Strategy

Run focused tests per milestone, then symbol searches, formatter check, `./mvnw test`, and finally ask the user to run `./mvnw clean verify` per repository guideline unless explicitly requested.

Actual validation results:

1. `./mvnw -Dtest=RuntimeSelectionTest test` passed with 3 tests.
2. `./mvnw -Dtest=FileSystemRuntimeExtensionSelectionRepositoryTest test` passed with 6 tests.
3. `./mvnw -Dtest=RuntimeExtensionInstallerTest,RuntimeExtensionModeEnablerTest,RuntimeExtensionModeDisablerTest,FileSystemRuntimeModeConfigurationRepositoryTest,FileSystemRuntimeExtensionSelectionRepositoryTest,Seed4JCliLauncherTest,RuntimeSelectionConfigurationTest,Seed4JCommandsFactoryTest,JavaProcessChildLauncherTest,ExtensionInstallCommandTest test` passed with 93 tests.
4. `./mvnw -Dtest=ExtensionInstallSpringContextTest test` passed with 1 test.
5. `rg -n "\bRuntimeExtensionConfiguration\b|\bRuntimeConfiguration\b|RuntimeSelection\.resolve|runtimeExtensionConfiguration\(" src/main/java src/test/java` returned no matches.
6. `npm run prettier:check` initially reported formatting drift in changed files; after `npm run prettier:format`, `npm run prettier:check` passed.
7. `./mvnw test` passed with 483 tests, 0 failures, 0 errors.
8. `rg -n "apply\(\) throws IOException|catch \(IOException" src/main/java/com/seed4j/cli/bootstrap/domain` still reports out-of-scope launcher/cache/loader/start-class IOException handling, not the active install/enable/selection flow.
9. `./mvnw clean verify` was not run automatically per repository guideline.

## Rollout and Recovery

Roll out as an internal refactor with no data migration. Reverting the PR is safe because the persisted active runtime layout remains unchanged.

## Lessons Learned

- `RuntimeSelection` is used by both parent process selection with a known JAR and child Spring binding without a JAR, so `extensionJarPath` remains optional.
- The old-symbol search must use exact word boundaries because `RuntimeConfiguration` is a substring of `InvalidRuntimeConfigurationException`.
- The remaining domain `Files`, `JarFile`, and `IOException` usage is in `Seed4JCliLauncher`, `RuntimeExtensionOverlayCache`, `RuntimeExtensionLoaderPathResolver`, `RuntimeExtensionStartClassResolver`, and `RuntimeExtensionCacheIdentityResolver`. Those classes are launcher/cache/loader concerns explicitly left out of this ExecPlan.
- The repository emits dependency convergence warnings during `./mvnw test`, but they were warnings in this run; the JUnit suite still passed.
- `_temporary/` is untracked in the worktree and unrelated to this refactor.
