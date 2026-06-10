# Migrate Launcher Coverage to Primary Bootstrap

This ExecPlan is a living document. Keep `Progress`, `Decisions`, `Risks`, and `Lessons Learned` up to date as work advances.

## Purpose / Big Picture

This change migrates bootstrap launcher coverage from the domain class `Seed4JCliLauncherTest` to the higher primary observation point `PreSpringBootstrapRunner.exitCodeFor(...)`. A user-visible bootstrap journey starts with command-line arguments entering the pre-Spring primary runner, so the tests should prove child runtime launch, local fallback, debug propagation, and pre-child failures from that boundary instead of protecting internal collaboration counts.

Safety boundary: This task is limited to authorized, defensive maintenance of this repository. Do not provide offensive guidance or policy-bypassing instructions.

## Scope

In scope: rename `ExtensionRuntimeBootstrapPrimaryTest` to `PreSpringBootstrapPrimaryTest`, keep its existing vertical extension runtime scenarios, add primary bootstrap behavior coverage migrated from `Seed4JCliLauncherTest`, and delete `Seed4JCliLauncherTest` if no stable domain contract remains there.

Out of scope: production code changes, public API changes, command behavior changes, and full `./mvnw clean verify`.

## Definitions

`PreSpringBootstrapRunner` is the primary adapter entry point used before Spring Boot starts.

`Child runtime` means a launched Java process request represented by `ChildRuntimeLaunchRequest`.

`Local CLI runner` means running the CLI in the current process, used for child mode and standard-mode fallback outside a packaged JAR.

`Packaged JAR` means the executable location is a regular `.jar` file accepted by `FileSystemPackagedExecutableDetector`.

## Existing Context

`src/test/java/com/seed4j/cli/bootstrap/infrastructure/primary/ExtensionRuntimeBootstrapPrimaryTest.java` already tests real extension runtime behavior through `PreSpringBootstrapRunner.exitCodeFor(...)`.

`src/test/java/com/seed4j/cli/bootstrap/domain/Seed4JCliLauncherTest.java` tests many bootstrap launcher decisions directly against `Seed4JCliLauncher`. Several assertions observe injected collaborator call counts or lower-level details that should be covered at primary bootstrap behavior level or by secondary infrastructure tests.

## Desired End State

`PreSpringBootstrapPrimaryTest` covers both existing extension runtime vertical scenarios and the migrated bootstrap primary journeys. Tests observe exit code, stderr/stdout, arguments, child launch request contents, local runner results, and debug diagnostics through `PreSpringBootstrapRunner.exitCodeFor(...)`. `Seed4JCliLauncherTest` is removed if it only contains migrated or intentionally redundant coverage.

## Milestones

### Milestone 1 - Prepare Primary Test Home

#### Goal

Rename the primary bootstrap test suite and keep existing vertical behavior green.

#### Changes

- [ ] Rename `src/test/java/com/seed4j/cli/bootstrap/infrastructure/primary/ExtensionRuntimeBootstrapPrimaryTest.java` to `PreSpringBootstrapPrimaryTest.java`.
- [ ] Rename class and internal class references used by helper methods.

#### Validation

- [ ] Command: `./mvnw -Dtest=PreSpringBootstrapPrimaryTest test`
- [ ] Expected result: renamed primary suite compiles and existing behavior passes.

#### Acceptance Criteria

- [ ] Existing version/list/apply/output/flat-jar scenarios still pass through `PreSpringBootstrapRunner`.

### Milestone 2 - Migrate Launcher Behaviors

#### Goal

Add primary bootstrap tests for child launch, local fallback, debug diagnostics, and pre-child failures.

#### Changes

- [ ] Add configurable primary runner helper assembling `PreSpringBootstrapRunner` with real filesystem repositories and packaged executable detector.
- [ ] Add recording child launcher, local runner, and diagnostics fakes that observe stable edge effects.
- [ ] Add test data helpers for config files, runtime artifacts, executable paths, and captured output.
- [ ] Add behavior tests for standard, extension, debug, local, child mode, invalid config, unreadable config, outside-JAR extension mode, and flat extension jar journeys.

#### Validation

- [ ] Command: `./mvnw -Dtest=PreSpringBootstrapPrimaryTest,RuntimeExtensionStartClassResolverTest,JavaProcessChildLauncherTest test`
- [ ] Expected result: migrated behavior passes from the primary observation point.

#### Acceptance Criteria

- [ ] Child launch requests expose expected runtime selection, arguments, executable path, extension jar, and debug value.
- [ ] Failure and fallback scenarios return observable exit codes and output without launching child runtime.

### Milestone 3 - Remove Redundant Domain Tests and Format

#### Goal

Delete domain tests that protect implementation topology after equivalent primary behavior coverage exists.

#### Changes

- [ ] Delete `src/test/java/com/seed4j/cli/bootstrap/domain/Seed4JCliLauncherTest.java` if no stable remaining contract remains.
- [ ] Format changed files with the repository formatter if needed.

#### Validation

- [ ] Command: `./mvnw -Dtest=PreSpringBootstrapPrimaryTest,RuntimeExtensionStartClassResolverTest,JavaProcessChildLauncherTest test`
- [ ] Command: `./mvnw test`
- [ ] Command: `npm run prettier:check`
- [ ] Expected result: all commands complete successfully.

#### Acceptance Criteria

- [ ] No references to `Seed4JCliLauncherTest` remain.
- [ ] Formatting check passes.

## Progress

- [x] ExecPlan created.
- [x] Milestone 1 started.
- [x] Milestone 1 completed.
- [x] Milestone 2 started.
- [x] Milestone 2 completed.
- [x] Milestone 3 started.
- [x] Milestone 3 completed.

## Decisions

- Decision: Treat the existing user-provided migration plan as intent, but re-read local production and test files before editing.
  Rationale: The repository may have changed since the plan was written, and tests should match current behavior.
  Date/Author: 2026-06-09 / Codex

- Decision: Do not add production changes for this migration.
  Rationale: The requested behavior already exists; the work is to move observation points and remove misleading coverage.
  Date/Author: 2026-06-09 / Codex

- Decision: Delete `Seed4JCliLauncherTest` after migrating stable journeys to `PreSpringBootstrapPrimaryTest`.
  Rationale: Remaining omitted cases were either collaboration-count assertions or secondary launcher/resolver behavior explicitly covered elsewhere.
  Date/Author: 2026-06-09 / Codex

## Risks and Mitigations

- Risk: New primary tests could still assert collaborator internals.
  Mitigation: Fakes only expose stable edge effects: launch request, exit code, arguments, stderr/stdout, and diagnostics activation.

- Risk: The primary suite could become slow if all scenarios run the real Spring app.
  Mitigation: Use recording child/local fakes for migrated launcher behaviors and keep existing real vertical extension runtime scenarios unchanged.

- Risk: File permissions may not simulate unreadable config reliably when tests run as a privileged user.
  Mitigation: Model unreadable config as a directory at the config file path, which consistently causes read failure through filesystem APIs.

## Validation Strategy

1. Run focused primary and affected secondary bootstrap tests.
2. Run `./mvnw test`.
3. Run `npm run prettier:check`.
4. Ask the user to run `./mvnw clean verify` locally if final full gate is needed.

## Rollout and Recovery

This is test-only. Recovery is to restore `Seed4JCliLauncherTest.java` and the previous primary test file name from version control if the migration hides a meaningful contract.

## Lessons Learned

- The migration should not force artificial RED for already implemented behavior; the useful failure mode here is compile/test failure from moving coverage to the correct boundary.
- `./mvnw -Dtest=PreSpringBootstrapPrimaryTest test` passed after the primary migration with 23 tests.
- `./mvnw -Dtest=PreSpringBootstrapPrimaryTest,RuntimeExtensionStartClassResolverTest,JavaProcessChildLauncherTest test` passed with 33 tests after deleting `Seed4JCliLauncherTest`.
- `./mvnw test` passed with 456 tests.
- `npm run prettier:check` initially failed on `PreSpringBootstrapPrimaryTest.java`; formatting that file with Prettier fixed the issue, and the check passed.
