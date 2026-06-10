# Refactor Pre-Spring Runtime Application Boundary

This ExecPlan is a living document. Keep `Progress`, `Decisions`, `Risks`, and `Lessons Learned` up to date as work advances.

## Purpose / Big Picture

This refactor keeps pre-Spring bootstrap behavior unchanged while making the application service receive only domain ports and business contracts. The observable behavior is that the CLI still chooses local, child, standard, and extension runtime paths exactly as before, but `PreSpringBootstrapApplicationService` no longer receives a fully constructed `Seed4JCliLauncher` or operational runtime values such as the executable JAR path and child process mode from callers.

Safety boundary: This task is limited to authorized, defensive maintenance of this repository. Do not provide offensive guidance or policy-bypassing instructions.

## Scope

In scope:

- Add a bootstrap domain port named `Seed4JCliRuntime`.
- Change `Seed4JCliLauncher` to depend on `Seed4JCliRuntime`.
- Change `PreSpringBootstrapApplicationService` to construct `Seed4JCliLauncher` from ports in its constructor.
- Keep `PreSpringBootstrapConfiguration` as the pre-Spring composition root that creates concrete secondary adapters.
- Add a secondary implementation that adapts `PreSpringRuntimeEnvironment` to `Seed4JCliRuntime`.
- Update focused tests without adding topology-only tests.

Out of scope:

- Removing `Path` from `ChildRuntimeLaunchRequest`.
- Changing CLI flags, output text, runtime selection files, or extension cache layout.
- Running `./mvnw clean verify`; repository instructions require asking the user to run it after focused checks.
- Touching untracked `_temporary/` content except this ExecPlan.

## Definitions

- Pre-Spring bootstrap: the code path that runs before Spring Boot is available and decides whether to run the CLI locally or launch a child Java process with extension runtime configuration.
- Domain port: a domain interface named by a business capability, implemented by secondary infrastructure.
- `Seed4JCliRuntime`: the new domain port that exposes the current CLI runtime facts needed by the launcher, namely executable JAR path and whether the process is already the child runtime.
- `PreSpringRuntimeEnvironment`: the current pre-Spring infrastructure input object containing CLI home, executable path, child mode, and Java executable path.

## Existing Context

`src/main/java/com/seed4j/cli/bootstrap/application/PreSpringBootstrapApplicationService.java` currently accepts a concrete `Seed4JCliLauncher`. `src/main/java/com/seed4j/cli/bootstrap/domain/Seed4JCliLauncher.java` currently accepts `Path executableJar` and `boolean childMode` directly. `src/main/java/com/seed4j/cli/bootstrap/composition/PreSpringBootstrapConfiguration.java` currently constructs the launcher in the composition root and passes it into the application service. Tests in `Seed4JCliAppTest`, `PreSpringBootstrapRunnerTest`, and `PreSpringBootstrapApplicationServiceTest` fabricate launchers only to instantiate higher-level objects.

## Desired End State

`PreSpringBootstrapApplicationService` has a constructor that receives `Seed4JCliRuntime`, `RuntimeModeConfigurationRepository`, `RuntimeExtensionSelectionRepository`, `ChildRuntimeLauncher`, `LocalCliRunner`, `PackagedExecutableDetector`, `BootstrapDiagnostics`, and `BootstrapOutput`, then constructs the launcher internally. `Seed4JCliLauncher` receives `Seed4JCliRuntime` and asks it for executable JAR and child runtime state. `PreSpringBootstrapConfiguration` creates secondary adapters, including a `PreSpringRuntimeEnvironmentSeed4JCliRuntime`, and passes only interfaces into the application service.

## Milestones

### Milestone 1 - Application Service Contract

#### Goal

Move service construction away from a concrete launcher while preserving launch behavior.

#### Changes

- [ ] Update `src/test/java/com/seed4j/cli/bootstrap/application/PreSpringBootstrapApplicationServiceTest.java` to construct the service with fakes for domain ports.
- [ ] Add `src/main/java/com/seed4j/cli/bootstrap/domain/Seed4JCliRuntime.java`.
- [ ] Update `PreSpringBootstrapApplicationService` to build `Seed4JCliLauncher` from the new constructor arguments.
- [ ] Update `Seed4JCliLauncher` to receive `Seed4JCliRuntime`.

#### Validation

- [ ] Command: `./mvnw -Dtest=PreSpringBootstrapApplicationServiceTest,Seed4JCliLauncherTest test`
- [ ] Expected result: both suites pass.

#### Acceptance Criteria

- [ ] `PreSpringBootstrapApplicationServiceTest` no longer imports or constructs `Seed4JCliLauncher`.
- [ ] Existing launcher behavior tests still prove child/local/extension launch behavior.

### Milestone 2 - Composition and Wrapper Tests

#### Goal

Keep pre-Spring composition as the only place that creates concrete adapters and remove launcher fabrication from wrapper tests.

#### Changes

- [ ] Add `src/main/java/com/seed4j/cli/bootstrap/infrastructure/secondary/PreSpringRuntimeEnvironmentSeed4JCliRuntime.java`.
- [ ] Update `src/main/java/com/seed4j/cli/bootstrap/composition/PreSpringBootstrapConfiguration.java`.
- [ ] Update `src/test/java/com/seed4j/cli/bootstrap/composition/PreSpringBootstrapConfigurationTest.java`.
- [ ] Update `src/test/java/com/seed4j/cli/bootstrap/infrastructure/primary/PreSpringBootstrapRunnerTest.java`.
- [ ] Update `src/test/java/com/seed4j/cli/Seed4JCliAppTest.java`.

#### Validation

- [ ] Command: `./mvnw -Dtest=PreSpringBootstrapApplicationServiceTest,PreSpringBootstrapRunnerTest,PreSpringBootstrapConfigurationTest,Seed4JCliAppTest,Seed4JCliLauncherTest test`
- [ ] Expected result: focused bootstrap suites pass.

#### Acceptance Criteria

- [ ] Composition creates concrete secondary adapters and passes only interfaces to the application service.
- [ ] Wrapper tests do not create a launcher just to instantiate the service.

### Milestone 3 - Formatting and Final Checks

#### Goal

Verify formatting and document validation status.

#### Changes

- [ ] Run repository formatter check.
- [ ] Update this ExecPlan with validation results and lessons.

#### Validation

- [ ] Command: `npm run prettier:check`
- [ ] Expected result: passes.

#### Acceptance Criteria

- [ ] Focused Maven tests pass.
- [ ] Prettier check passes.
- [ ] User is asked to run `./mvnw clean verify` locally.

## Progress

- [x] Milestone 1 started
- [x] Milestone 1 completed
- [x] Milestone 2 started
- [x] Milestone 2 completed
- [x] Milestone 3 started
- [x] Milestone 3 completed

## Decisions

- Decision: Use `Seed4JCliRuntime` as a domain port even though it returns a `Path`.
  Rationale: The executable JAR path is still part of the launcher contract and `ChildRuntimeLaunchRequest` remains out of scope; this change specifically removes operational path and child mode from the application service boundary.
  Date/Author: 2026-06-08 / Codex

## Risks and Mitigations

- Risk: Tests could become topology tests that merely assert the application service constructs a launcher.
  Mitigation: Keep tests observing launch outcomes through `exitCodeFor` or existing public launcher behavior.
- Risk: Moving construction into the application service could obscure pre-Spring concrete wiring.
  Mitigation: Keep `PreSpringBootstrapConfiguration` explicit and only pass domain/application contracts into the service.

## Validation Strategy

1. Run focused application and launcher tests after changing the service and launcher contract.
2. Run focused pre-Spring wrapper/composition tests after rewiring composition.
3. Run `npm run prettier:check`.
4. Ask the user to run `./mvnw clean verify` locally.

Validation results on 2026-06-08:

- `./mvnw -Dtest=PreSpringBootstrapApplicationServiceTest test` failed as expected during RED because `Seed4JCliRuntime` and the new application service constructor did not exist.
- `./mvnw -Dtest=PreSpringBootstrapApplicationServiceTest,PreSpringBootstrapRunnerTest,PreSpringBootstrapConfigurationTest,Seed4JCliAppTest,Seed4JCliLauncherTest test` passed with 36 tests.
- `./mvnw -Dtest=Seed4JCliLauncherConstructionTest,ExtensionRuntimeBootstrapInProcessTest test` passed with 7 tests because those suites also compile against the launcher constructor.
- After final formatting and constructor validation edits, `./mvnw -Dtest=PreSpringBootstrapApplicationServiceTest,PreSpringBootstrapRunnerTest,PreSpringBootstrapConfigurationTest,Seed4JCliAppTest,Seed4JCliLauncherTest,Seed4JCliLauncherConstructionTest,ExtensionRuntimeBootstrapInProcessTest test` passed with 43 tests.
- `npm run prettier:check` passed.

## Rollout and Recovery

This is an internal architecture refactor with unchanged CLI behavior. If a regression appears, revert the refactor commit and restore the previous launcher constructor and composition wiring, then rerun the focused bootstrap tests.

## Lessons Learned

- `PreSpringBootstrapApplicationServiceTest` could stay at the observable application-service path by using simple fakes for ports instead of constructing a launcher.
- Updating `Seed4JCliLauncher` constructor touched additional domain tests outside the original focused command list; `Seed4JCliLauncherConstructionTest` and `ExtensionRuntimeBootstrapInProcessTest` were included in validation because they compile against that public constructor.
