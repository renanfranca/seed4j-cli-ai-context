# Enforce Command and Bootstrap Boundary

This ExecPlan is a living document. Keep `Progress`, `Decisions`, `Risks`, and `Lessons Learned` up to date as work advances.

## Purpose / Big Picture

The `command` bounded context currently reaches into `bootstrap.application` and `bootstrap.domain` from primary CLI classes. This refactor makes the boundary explicit: command primary adapters call command application services and command ports, while cross-context runtime work flows through bootstrap Java primary adapters. Users should observe the same CLI behavior for `seed4j extension install` and `seed4j --version`, with architecture tests preventing the old dependency shape from returning.

Safety boundary: This task is limited to authorized, defensive maintenance of this repository. Do not provide offensive guidance or policy-bypassing instructions.

## Scope

In scope:

- Tighten `HexagonalArchTest` so bounded contexts and shared kernels cannot depend on another context's application layer.
- Remove the special command exception that allowed command primary adapters to depend directly on bootstrap domain.
- Add bootstrap `infrastructure.primary.Java*` adapters as the public Java boundary for command secondary adapters.
- Add command-owned domain types, ports, and application services for runtime extension installation and runtime display.
- Keep existing CLI output and persisted installation behavior.

Out of scope:

- Changing CLI command names, flags, or output text except where unavoidable to preserve current behavior.
- Running `./mvnw clean verify`; repository guidance says to ask the user to run it after focused checks pass.
- Refactoring unrelated bootstrap internals.

## Definitions

- Bounded context: a package marked by `@BusinessContext`, such as `com.seed4j.cli.command` or `com.seed4j.cli.bootstrap`.
- Primary adapter: code under `..infrastructure.primary..` that exposes a context to an external caller such as Picocli or another Java context.
- Secondary adapter: code under `..infrastructure.secondary..` that implements a domain port and talks to an external mechanism or another context's primary adapter.
- Java primary adapter: a bootstrap primary adapter with a simple class name starting with `Java`; this is the allowed Java-facing entry point for another context.
- Command runtime display: command-owned representation of runtime mode and optional distribution metadata used by `seed4j --version`.

## Existing Context

`src/main/java/com/seed4j/cli/command/infrastructure/primary/ExtensionInstallCommand.java` depends on `RuntimeExtensionApplicationService` from `bootstrap.application` and constructs bootstrap domain types directly. `src/main/java/com/seed4j/cli/command/infrastructure/primary/Seed4JCommandsFactory.java` depends on bootstrap `RuntimeSelection`, `RuntimeMode`, and distribution value objects to render version output. `src/test/java/com/seed4j/cli/HexagonalArchTest.java` currently has a special rule allowing command primary adapters to depend on other bounded context domains.

## Desired End State

`command` code depends only on command-owned application/domain contracts and shared kernel types. Runtime extension installation is requested through a command application service and command domain port. A command secondary adapter calls `bootstrap.infrastructure.primary.JavaRuntimeExtensionInstaller`, which translates strings and paths into bootstrap domain types internally. Runtime display for `seed4j --version` is requested through a command query/port and a command secondary adapter calls a bootstrap Java primary adapter. The architecture test passes and forbids reintroducing direct command dependencies on bootstrap domain or application.

## Milestones

### Milestone 1 - Tighten Architecture Rules

#### Goal

Make the architecture test fail against current command-to-bootstrap dependencies before production code changes.

#### Changes

- [x] Edit `src/test/java/com/seed4j/cli/HexagonalArchTest.java` to remove the command exception for other bounded context domains.
- [x] Add a rule forbidding bounded contexts and shared kernels from depending on another bounded context's `..application..`.
- [x] Add a rule that `..infrastructure.primary..` Spring components with simple names starting with `Java` may only be depended on by `..infrastructure.secondary..`.

#### Validation

- [x] Command: `./mvnw -Dtest=HexagonalArchTest test`
- [x] Expected result: fails on existing command dependencies to bootstrap domain/application.

#### Acceptance Criteria

- [x] The failure clearly names command classes as violating the tightened boundary.

### Milestone 2 - Add Bootstrap Java Primary Adapters and Command Ports

#### Goal

Create stable cross-context Java contracts without exposing bootstrap domain types to command.

#### Changes

- [x] Add `JavaRuntimeExtensionInstaller` in `bootstrap.infrastructure.primary`.
- [x] Add command domain install request/result/error concepts and an install port.
- [x] Add command runtime display domain type and query port.
- [x] Add command secondary adapters that implement those ports by calling bootstrap Java primary adapters.

#### Validation

- [x] Command: `./mvnw -Dtest=HexagonalArchTest test`
- [x] Expected result: remaining failures identify command primary classes until they are rewired.

#### Acceptance Criteria

- [x] New command code does not import `bootstrap.domain` or `bootstrap.application`.

### Milestone 3 - Rewire Command Primary Adapters

#### Goal

Make Picocli primary adapters depend on command application services only.

#### Changes

- [x] Change `ExtensionInstallCommand` to call command application and handle command errors.
- [x] Change `Seed4JCommandsFactory` to use command-owned runtime display query.
- [x] Update Spring wiring through component discovery without manual cross-layer shortcuts.

#### Validation

- [x] Command: `./mvnw -Dtest=HexagonalArchTest test`
- [x] Expected result: passes.

#### Acceptance Criteria

- [x] `rg "bootstrap\\.(application|domain)" src/main/java/com/seed4j/cli/command` returns no production imports.

### Milestone 4 - Update Behavior Tests

#### Goal

Keep existing CLI behavior while removing bootstrap construction from command tests.

#### Changes

- [x] Update `ExtensionInstallCommandTest` and `CliFixture` to construct command application paths.
- [x] Update `ExtensionInstallSpringContextTest` context filters for new adapter classes.
- [x] Update `Seed4JCommandsFactoryTest` to use command-owned runtime display concepts.

#### Validation

- [x] Command: `./mvnw -Dtest=ExtensionInstallCommandTest test`
- [x] Command: `./mvnw -Dtest=ExtensionInstallSpringContextTest test`
- [x] Command: `./mvnw -Dtest=Seed4JCommandsFactoryTest test`
- [x] Expected result: all pass.

#### Acceptance Criteria

- [x] Extension install still reports success and writes config, jar, and metadata as before.
- [x] Version output still reports standard or extension mode and extension distribution metadata.

## Progress

- [x] Milestone 1 started
- [x] Milestone 1 completed
- [x] Milestone 2 started
- [x] Milestone 2 completed
- [x] Milestone 3 started
- [x] Milestone 3 completed
- [x] Milestone 4 started
- [x] Milestone 4 completed

## Decisions

- Decision: Use command-owned value objects instead of passing bootstrap domain types through command.
  Rationale: The repository guidance calls for Types Driven Development and forbids hidden bootstrap operational concepts from leaking across bounded contexts.
  Date/Author: 2026-06-08 / Codex
- Decision: Split runtime extension installation and runtime display into separate command application services.
  Rationale: Installation is a command use case, while version display is a query; coupling them would create a wider command API than the CLI needs.
  Date/Author: 2026-06-08 / Codex

## Risks and Mitigations

- Risk: The Java primary adapter naming rule could accidentally constrain ordinary Picocli primary classes if applied too broadly.
  Mitigation: Scope the rule to classes residing in `..infrastructure.primary..`, annotated with Spring `@Component`, and with simple names starting with `Java`.
- Risk: Tests might be rewritten around new production topology instead of behavior.
  Mitigation: Keep extension install and version tests at the Picocli/user-facing observation point.

## Validation Strategy

1. Run `./mvnw -Dtest=HexagonalArchTest test` after architecture rule changes and after rewiring.
2. Run focused command behavior tests: `ExtensionInstallCommandTest`, `ExtensionInstallSpringContextTest`, and `Seed4JCommandsFactoryTest`.
3. Run `npm run prettier:check` or `npm run prettier:format` for changed Java and Markdown files if formatting is needed.
4. Ask the user to run `./mvnw clean verify` locally after focused checks pass.

Validation results on 2026-06-08:

- `./mvnw -Dtest=HexagonalArchTest test` first failed as expected on `ExtensionInstallCommand` and `Seed4JCommandsFactory` direct bootstrap dependencies.
- `./mvnw -Dtest=HexagonalArchTest,ExtensionInstallCommandTest,ExtensionInstallSpringContextTest,Seed4JCommandsFactoryTest test` passed with 44 tests.
- `npm run prettier:check` passed.
- `rg "bootstrap\\.(application|domain)" src/main/java/com/seed4j/cli/command` and the same scan over command primary tests returned no matches.

## Rollout and Recovery

This is an internal refactor with unchanged CLI behavior. If the change causes regressions, revert the commit containing the boundary refactor and restore the previous direct command-to-bootstrap wiring while keeping the ExecPlan for follow-up.

## Lessons Learned

- Tightening `HexagonalArchTest` produced the expected RED failure: command primary classes depended on both `bootstrap.domain` and `bootstrap.application`.
- Command primary tests should not manually assemble bootstrap application services; fake command ports cover Picocli behavior, while `ExtensionInstallSpringContextTest` covers the real Spring-managed persistence graph.
