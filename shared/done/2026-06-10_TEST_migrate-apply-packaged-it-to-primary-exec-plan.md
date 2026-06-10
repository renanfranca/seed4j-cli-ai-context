# Migrate Apply Packaged IT To Primary Coverage

This ExecPlan is a living document. Keep `Progress`, `Decisions`, `Risks`, and `Lessons Learned` up to date as work advances.

## Purpose / Big Picture

Move detailed `apply` behavior coverage for extension runtime resources from the slow packaged JAR integration test into the faster primary bootstrap test. Users should still have confidence that the real packaged CLI can launch extension mode, but exact resource precedence behavior should be observed through `PreSpringBootstrapRunner` in process.

## Scope

In scope: strengthen `PreSpringBootstrapPrimaryTest`, add a primary behavior test for standard versus extension common dependency collision, and reduce the packaged JAR IT to one smoke test. Out of scope: production API changes, tests for internal classloader/resolver/cache helpers, and running `./mvnw clean verify` automatically.

## Definitions

`PreSpringBootstrapRunner` is the primary adapter entry point used before Spring starts. `Extension runtime` means a configured active runtime extension installed under the test user's `~/.config/seed4j-cli/runtime/active`. `Packaged smoke` means a minimal `java -jar` test that proves the built CLI JAR starts and applies an extension module in a separate JVM.

## Existing Context

`src/test/java/com/seed4j/cli/bootstrap/infrastructure/primary/PreSpringBootstrapPrimaryTest.java` already exercises extension `apply` through an in-process child runtime launcher but only asserts that output files exist. `src/test/java/com/seed4j/cli/bootstrap/domain/ExtensionRuntimeBootstrapApplyPackagedJarIT.java` currently contains two detailed packaged JAR tests for common dependency collision and shared runtime resources.

## Desired End State

Primary tests assert exact generated behavior: overridden Prettier version and template marker where appropriate. The packaged JAR test is renamed/moved to `bootstrap/infrastructure/primary` and keeps only a single smoke scenario proving the real `java -jar` extension apply flow exits successfully and writes basic artifacts.

## Milestones

### Milestone 1 - Strengthen Primary Apply Coverage

#### Goal

Observe detailed extension resource behavior through the primary runner.

#### Changes

- [ ] Edit `src/test/java/com/seed4j/cli/bootstrap/infrastructure/primary/PreSpringBootstrapPrimaryTest.java` to assert `package.json` contains `"prettier": "3.6.2"` and `.prettierrc` contains `seed4j-extension-template-override`.
- [ ] Add a primary behavior test proving standard mode does not receive the overridden Prettier version while extension mode does for `generator/dependencies/common/package.json`.

#### Validation

- [ ] Command: `./mvnw -Dtest=PreSpringBootstrapPrimaryTest test`
- [ ] Expected result: test suite exits 0.

#### Acceptance Criteria

- [ ] Detailed resource precedence behavior is covered without `java -jar`.

### Milestone 2 - Reduce Packaged IT To Smoke

#### Goal

Keep only process and packaging confidence in Failsafe.

#### Changes

- [ ] Replace `ExtensionRuntimeBootstrapApplyPackagedJarIT` with `ExtensionRuntimeBootstrapApplyPackagedJarSmokeIT` under `src/test/java/com/seed4j/cli/bootstrap/infrastructure/primary`.
- [ ] Remove exact Prettier/template assertions and standard versus extension comparison from the packaged test.

#### Validation

- [ ] Command: `./mvnw -Dit.test=ExtensionRuntimeBootstrapApplyPackagedJarSmokeIT failsafe:integration-test failsafe:verify`
- [ ] Expected result: smoke IT exits 0, assuming a packaged CLI JAR exists in `target/`.

#### Acceptance Criteria

- [ ] Packaged smoke proves `java -jar` with extension mode can apply an extension module and write basic artifacts.

## Progress

- [x] ExecPlan created
- [x] Milestone 1 started
- [x] Milestone 1 completed
- [x] Milestone 2 started
- [x] Milestone 2 completed
- [x] Focused validation completed

## Decisions

- Decision: Treat the provided migration plan as the source of user intent.
  Rationale: The user explicitly supplied it from a previous agent and asked for implementation in fresh context.
  Date/Author: 2026-06-10 / Codex
- Decision: Keep detailed Prettier version and template marker assertions in the primary runner test.
  Rationale: Those are observable generated files and no longer need the real packaged JVM once the primary harness models extension resource precedence.
  Date/Author: 2026-06-10 / Codex

## Risks and Mitigations

- Risk: Primary tests might accidentally assert internal classloader implementation details.
  Mitigation: Assert only generated files and CLI launch results through `PreSpringBootstrapRunner`.
- Risk: Packaged smoke can fail when no packaged JAR exists in `target/`.
  Mitigation: Run the requested Failsafe command and report exact failure if packaging is missing.

## Validation Strategy

1. Run `./mvnw -Dtest=PreSpringBootstrapPrimaryTest test` after primary test changes.
2. Run `./mvnw -Dit.test=ExtensionRuntimeBootstrapApplyPackagedJarSmokeIT failsafe:integration-test failsafe:verify` after reducing the IT.
3. Ask the user to run `./mvnw clean verify` locally for the final repository gate.

Validation completed on 2026-06-10:

- `./mvnw -Dtest=PreSpringBootstrapPrimaryTest test` exited 0.
- `./mvnw -Dit.test=ExtensionRuntimeBootstrapApplyPackagedJarSmokeIT failsafe:integration-test failsafe:verify` exited 0.
- `npm run prettier:check` exited 0.

## Rollout and Recovery

This is test-only. Recovery is reverting the modified test files and removing the new smoke test file.

## Lessons Learned

- `FileSystemProjectFiles` reads resources through `FileSystemProjectFiles.class.getResource...`; the primary in-process child classloader had to load that class child-first so extension overlay resources are visible like they are under the real packaged launcher.
