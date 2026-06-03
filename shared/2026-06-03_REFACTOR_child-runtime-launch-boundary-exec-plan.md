# Harden Child Runtime Launch Boundary

This ExecPlan is a living document. Keep `Progress`, `Decisions`, `Risks`, and `Lessons Learned` up to date as work advances.

## Purpose / Big Picture

Move Java/Spring child-process protocol details out of bootstrap domain while preserving runtime launch behavior. `Seed4JCliLauncher` should decide whether to run locally or launch child runtime, and secondary infrastructure should translate that decision into JVM command details. Also remove the unused short constructor from `Seed4JCliLauncher`.

Safety boundary: This task is limited to authorized, defensive maintenance of this repository.

## Scope

In scope: refactor the child runtime launch port/request, move technical system-property/main-class assembly to secondary, update tests, remove the short constructor.

Out of scope: changing runtime mode semantics, changing `loader.path` contents, changing exit codes, or running `./mvnw clean verify` automatically.

## Definitions

`Child runtime launch request` means a domain-level intention to launch the CLI child runtime with a `RuntimeSelection`, executable JAR, CLI args, and debug flag.

`Java child process request` means the secondary-only technical representation containing JVM main class and system properties.

## Existing Context

`Seed4JCliLauncher` currently builds `JavaChildProcessRequest` and sets technical names such as `PropertiesLauncher`, `loader.path`, `logging.config`, and `spring.main.log-startup-info`.

`JavaProcessChildLauncher` already lives in `bootstrap/infrastructure/secondary`, but it receives a DTO from domain that is too Java/Spring-specific.

There is a short package-private `Seed4JCliLauncher` constructor that is no longer used by production wiring.

## Desired End State

Domain owns runtime decisions only. Secondary owns JVM/Spring launch protocol translation and command assembly. The short constructor is gone. Runtime extension launch behavior remains byte-for-byte equivalent for command/system-property tests.

## Milestones

### Milestone 1 - Save ExecPlan And Remove Legacy Constructor

#### Goal

Record this plan and remove dead constructor surface.

#### Changes

- [ ] Save this ExecPlan at the plan file path above.
- [ ] Remove the short constructor from `Seed4JCliLauncher`.
- [ ] Update `Seed4JCliLauncherFactoryTest` so it no longer expects the removed constructor.

#### Validation

- [ ] Command: `./mvnw -Dtest=Seed4JCliLauncherFactoryTest test`
- [ ] Expected result: passes.

#### Acceptance Criteria

- [ ] No production path or test uses the short constructor.

### Milestone 2 - Introduce Domain Launch Intent

#### Goal

Make the child launch port receive domain intent instead of Java/Spring details.

#### Changes

- [ ] Replace `ChildProcessLauncher.launch(JavaChildProcessRequest)` with `ChildRuntimeLauncher.launch(ChildRuntimeLaunchRequest)`.
- [ ] Create `ChildRuntimeLaunchRequest` in domain with `Path executableJar`, `RuntimeSelection runtimeSelection`, `Seed4JCliArguments arguments`, and `boolean debug`.
- [ ] Update `Seed4JCliLauncher` to call the new port and stop building system properties.

#### Validation

- [ ] Command: `./mvnw -Dtest=Seed4JCliLauncherTest,Seed4JCliLauncherFactoryTest test`
- [ ] Expected result: domain tests pass by asserting launch intent, not technical JVM properties.

#### Acceptance Criteria

- [ ] `Seed4JCliLauncher` no longer contains `PropertiesLauncher`, `loader.path`, `logging.config`, or `spring.main.log-startup-info`.

### Milestone 3 - Move JVM/Spring Translation To Secondary

#### Goal

Preserve exact child command behavior in infrastructure.

#### Changes

- [ ] Move `JavaChildProcessRequest` to secondary or make it private/internal to `JavaProcessChildLauncher`.
- [ ] Inject `RuntimeExtensionStartClassResolver`, `RuntimeExtensionOverlayCache`, and `RuntimeExtensionLoaderPathResolver` into `JavaProcessChildLauncher`.
- [ ] Build all JVM system properties and `PropertiesLauncher` main class inside `JavaProcessChildLauncher`.
- [ ] Keep sorted `-D` ordering, `-cp`, executable JAR, main class, and CLI args unchanged.

#### Validation

- [ ] Command: `./mvnw -Dtest=JavaProcessChildLauncherTest,ExtensionRuntimeBootstrapInProcessTest test`
- [ ] Expected result: secondary command construction and in-process extension runtime checks pass.

#### Acceptance Criteria

- [ ] Covered child process commands remain equivalent to current behavior.
- [ ] Extension mode still resolves start class, overlay cache, and loader path before launching.

### Milestone 4 - Recompose And Verify Architecture

#### Goal

Wire new port shape through pre-Spring composition.

#### Changes

- [ ] Update `Seed4JCliLauncherFactory.LauncherDependencies`.
- [ ] Update `PreSpringBootstrapConfiguration`.
- [ ] Update primary/application tests that construct launcher dependencies.

#### Validation

- [ ] Command: `./mvnw -Dtest=HexagonalArchTest,PreSpringBootstrapConfigurationTest,PreSpringBootstrapApplicationServiceTest,Seed4JCliAppTest test`
- [ ] Expected result: architecture and public bootstrap paths pass.

#### Acceptance Criteria

- [ ] Composition remains the only place wiring secondary adapters into bootstrap flow.
- [ ] Domain has no direct Java/Spring child-launch protocol strings.

### Milestone 5 - Broad Validation

#### Goal

Confirm behavior and formatting.

#### Changes

- [ ] Run focused moved/refactored package tests.
- [ ] Run unit suite.
- [ ] Run formatter check.
- [ ] Update ExecPlan validation outcomes.

#### Validation

- [ ] Command: `./mvnw test`
- [ ] Expected result: unit/component tests pass.
- [ ] Command: `npm run prettier:check`
- [ ] Expected result: formatted files pass.

#### Acceptance Criteria

- [ ] `HexagonalArchTest` passes.
- [ ] Runtime extension launch tests pass.
- [ ] User is asked to run `./mvnw clean verify`.

## Progress

- [x] ExecPlan saved
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

- Decision: Use `seed4j-execplan-tdd`, `implement-execplan`, and `tdd-strict-autonomous-quiet`.
  Rationale: The change is architecture-sensitive and should progress through small behavior-preserving TDD cycles.
  Date/Author: 2026-06-03 / Codex
- Decision: Keep domain `Path executableJar` in the launch intent.
  Rationale: It represents the current executable artifact and is already accepted as a boundary value.
  Date/Author: 2026-06-03 / Codex

## Risks and Mitigations

- Risk: Moving system-property assembly can change exact child process behavior.
  Mitigation: Assert exact command and property values in secondary tests.
- Risk: Domain tests may become too implementation-oriented.
  Mitigation: Domain tests assert runtime launch intent only.
- Risk: `JavaChildProcessRequest` may remain publicly visible by accident.
  Mitigation: Move it to secondary or make it adapter-internal.

## Validation Strategy

Use quiet autonomous TDD. Each cycle starts with one failing behavior, runs the full relevant focused suite, implements the minimum code, and reruns the suite. Run a public-path checkpoint at least every two cycles. Finish with `./mvnw test` and `npm run prettier:check`.

## Rollout and Recovery

This is an internal refactor with no intended CLI behavior change. If launch behavior regresses, revert the branch or restore the previous `JavaProcessChildLauncher`/port shape while keeping the constructor cleanup if independently green.

## Lessons Learned

- `Seed4JCliLauncher` is not unused; production reaches it through the factory and pre-Spring composition.
- The short constructor is unused legacy surface after port extraction.
- The important boundary issue is not command execution in domain, but Java/Spring launch protocol strings being assembled before crossing to secondary.
- `./mvnw -Dtest=Seed4JCliLauncherFactoryTest test` passed after removing the short constructor and updating the constructor visibility assertion.
- Missing extension `Start-Class` validation moves from `Seed4JCliLauncher` to the secondary child launcher because start-class resolution is part of JVM/Spring launch translation, not domain runtime selection.
- `./mvnw -Dtest=Seed4JCliLauncherTest,Seed4JCliLauncherFactoryTest test` passed after replacing Java/Spring request assertions with child runtime launch intent assertions.
- `./mvnw -Dtest=JavaProcessChildLauncherTest,ExtensionRuntimeBootstrapInProcessTest test` passed after moving JVM/Spring property assembly to `JavaProcessChildLauncher`.
- `./mvnw -Dtest=HexagonalArchTest,PreSpringBootstrapConfigurationTest,PreSpringBootstrapApplicationServiceTest,Seed4JCliAppTest test` passed after recomposing `JavaProcessChildLauncher` with secondary launch translators.
- `./mvnw test` passed with 493 tests, 0 failures, 0 errors, 0 skipped.
- `npm run prettier:check` passed after formatting `ExtensionRuntimeBootstrapInProcessTest.java` and `Seed4JCliLauncherTest.java`.
- Follow-up cleanup removed unused `Seed4JCliHome cliHome` from `Seed4JCliLauncher`; `./mvnw -Dtest=Seed4JCliLauncherTest,Seed4JCliLauncherFactoryTest,ExtensionRuntimeBootstrapInProcessTest test` and `npm run prettier:check` passed.
