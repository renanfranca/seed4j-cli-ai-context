# Test Bootstrap Primary Extension Runtime Journeys

This ExecPlan is a living document. Keep `Progress`, `Decisions`, `Risks`, and `Lessons Learned` up to date as work advances.

## Purpose / Big Picture

This change moves extension runtime bootstrap journey tests to the CLI bootstrap primary entry point, `PreSpringBootstrapRunner.exitCodeFor(...)`. The user-visible behavior is that extension mode works through the same pre-Spring runner used by packaged startup: version output, module listing, resource precedence, logging cleanup, apply behavior, and invalid extension jar rejection are all observable through CLI arguments and filesystem output. External `java -jar` tests remain as smoke tests for packaged wiring, while broad behavior coverage runs in-process so JaCoCo can observe it.

## Scope

In scope:

- Add primary-entry behavior tests in `src/test/java/com/seed4j/cli/bootstrap/infrastructure/primary/ExtensionRuntimeBootstrapPrimaryTest.java`.
- Move `ExtensionRuntimeFixture` from the bootstrap domain test package to a shared bootstrap fixture package.
- Replace broad in-process domain journey tests with primary-runner tests using real filesystem adapters and a test-only in-process child runtime launcher.
- Reduce packaged extension runtime integration tests to smoke-level coverage for real jar execution.
- Validate with focused Maven and formatting commands.

Out of scope:

- Production public API changes.
- Full `./mvnw clean verify`, which remains a user-run final gate per repository instructions.
- Mockito-based collaborator call assertions for these journeys.

## Definitions

- `PreSpringBootstrapRunner`: the primary adapter entry point that receives raw CLI arguments before Spring starts.
- `Extension mode`: runtime mode where the CLI uses an installed extension jar from the active runtime directory under the CLI home.
- `In-process child runtime launcher`: a test adapter implementing `ChildRuntimeLauncher` that applies the same child runtime system properties and runs the Spring local CLI in the same JVM. It exists because a real `java -jar` child process is not useful for JaCoCo behavior coverage.
- `Packaged smoke test`: a narrow integration test that executes the packaged jar only to prove real jar wiring still starts and fails in critical cases.

## Existing Context

`src/test/java/com/seed4j/cli/bootstrap/domain/ExtensionRuntimeBootstrapInProcessTest.java` currently exercises extension runtime journeys by constructing `Seed4JCliLauncher` directly in the domain test package. That is below the intended primary observation point and duplicates behavior now better observed through `PreSpringBootstrapRunner.exitCodeFor(...)`.

`src/test/java/com/seed4j/cli/bootstrap/domain/ExtensionRuntimeFixture.java` builds temporary extension jars and installs runtime config under a temporary user home. It is reusable fixture code but currently lives in the domain package with package-private install methods.

Packaged integration tests under `src/test/java/com/seed4j/cli/bootstrap/domain/*PackagedJarIT.java` currently cover both real-process wiring and broad list/apply/version behavior. Broad behavior should move to in-process primary tests; packaged tests should remain narrow.

## Desired End State

Primary behavior tests run through `PreSpringBootstrapRunner.exitCodeFor(...)` with real runtime repositories, validators, Spring local runner, overlay cache, loader path resolver, and captured output. Only the child process boundary is replaced by an in-process launcher for coverage.

The fixture lives under a shared bootstrap test fixture package and existing fixture tests/imports are updated. Packaged tests are smoke tests only. Production code remains unchanged unless a blocking design issue is discovered and recorded first.

## Milestones

### Milestone 1 - Shared Fixture

#### Goal

Make the extension runtime fixture reusable from primary bootstrap tests without placing tests in the domain package.

#### Changes

- [ ] Move `src/test/java/com/seed4j/cli/bootstrap/domain/ExtensionRuntimeFixture.java` to `src/test/java/com/seed4j/cli/bootstrap/fixture/ExtensionRuntimeFixture.java`.
- [ ] Update the package declaration and visibility for fixture install methods and the returned paths record.
- [ ] Update existing tests to import the new fixture package.

#### Validation

- [ ] Command: `./mvnw -Dtest=ExtensionRuntimeFixtureTest test`
- [ ] Expected result: fixture tests compile and pass.

#### Acceptance Criteria

- [ ] Fixture tests still prove valid extension runtime artifacts are installed.
- [ ] Primary package tests can call the fixture without package leakage.

### Milestone 2 - Primary Runner Journeys

#### Goal

Replace domain-level in-process journey tests with behavior tests at `PreSpringBootstrapRunner.exitCodeFor(...)`.

#### Changes

- [ ] Add `src/test/java/com/seed4j/cli/bootstrap/infrastructure/primary/ExtensionRuntimeBootstrapPrimaryTest.java`.
- [ ] Compose `PreSpringBootstrapApplicationService` in test with real filesystem repositories, validators, Spring local runner, overlay cache, start class resolver, loader path resolver, packaged executable detector, diagnostics, and bootstrap output.
- [ ] Add an in-process `ChildRuntimeLauncher` test adapter that applies child runtime system properties and delegates to `LocalCliRunner`.
- [ ] Delete or reduce `ExtensionRuntimeBootstrapInProcessTest` once equivalent primary behavior exists.

#### Validation

- [ ] Command: `./mvnw -Dtest=ExtensionRuntimeBootstrapPrimaryTest test`
- [ ] Expected result: primary journey tests pass.

#### Acceptance Criteria

- [ ] `--version` in extension mode prints runtime mode, distribution id, and distribution version through the primary runner.
- [ ] `list` keeps the standard catalog, adds only the extension-only slug, has no duplicates, and supports custom-package extension slugs.
- [ ] Extension hidden-resource overrides do not hide core modules such as `gradle-java`.
- [ ] Extension logging/application overrides do not leak Spring banner, startup logs, extension markers, or Logback scan warnings.
- [ ] `apply init` plus `apply prettier` uses the extension common-source override.
- [ ] Shared-runtime extension apply module writes the expected `package.json` and `.prettierrc`.
- [ ] Flat extension jar fails before child runtime and reports `BOOT-INF/classes`.

### Milestone 3 - Packaged Smoke Tests

#### Goal

Keep external process coverage only where it proves real packaged jar wiring.

#### Changes

- [ ] Trim packaged list/apply/version tests so they no longer duplicate broad primary behavior.
- [ ] Keep smoke coverage for successful extension `--version` through the packaged jar and flat jar failure before child runtime.

#### Validation

- [ ] Command: `./mvnw -DskipTests package`
- [ ] Command: `./mvnw -Dit.test='*Bootstrap*PackagedJarIT,*Bootstrap*ListPackagedJarIT' failsafe:integration-test failsafe:verify`
- [ ] Expected result: packaged smoke tests pass after a jar exists.

#### Acceptance Criteria

- [ ] Packaged tests verify real jar execution but do not carry broad behavior assertions now covered by primary tests.

### Milestone 4 - Broader Targeted Validation

#### Goal

Confirm bootstrap tests and formatting remain healthy without running the full local gate automatically.

#### Changes

- [ ] Update imports and formatting after test movement.

#### Validation

- [ ] Command: `./mvnw -Dtest='*Bootstrap*Test' test`
- [ ] Command: `npm run prettier:check`
- [ ] Expected result: targeted bootstrap tests and formatting pass.

#### Acceptance Criteria

- [ ] All changed test surfaces pass focused validation.
- [ ] User is asked to run `./mvnw clean verify` locally as the final gate.

## Progress

- [x] ExecPlan created.
- [x] Milestone 1 started.
- [x] Milestone 1 completed.
- [x] Milestone 2 started.
- [x] Milestone 2 completed.
- [x] Milestone 3 started.
- [x] Milestone 3 completed.
- [x] Milestone 4 started.
- [x] Milestone 4 completed.

## Validation Results

- `./mvnw -Dtest=ExtensionRuntimeFixtureTest test` passed.
- `./mvnw -Dtest=ExtensionRuntimeBootstrapPrimaryTest test` passed.
- `./mvnw -Dtest='*Bootstrap*Test' test` passed.
- `./mvnw -DskipTests package` passed.
- `./mvnw test-compile -Dit.test='*Bootstrap*PackagedJarIT,*Bootstrap*ListPackagedJarIT' failsafe:integration-test failsafe:verify` passed.
- `npm run prettier:check` passed.

## Decisions

- Decision: Use primary runner tests with a test-only in-process child launcher instead of production API changes.
  Rationale: This preserves production boundaries while observing the real pre-Spring entry point and keeping coverage in the same JVM.
  Date/Author: 2026-06-09 / Codex

- Decision: Use a test Spring Boot application class that scans production packages but excludes fixture-only extension packages, and set a child runtime context classloader that includes the extension overlay and jar.
  Rationale: Test extension classes live under `com.seed4j.cli` and would otherwise be discovered by normal component scanning; `loader.path` is interpreted by the packaged Boot launcher, so same-JVM child tests need an explicit context classloader to observe extension resources.
  Date/Author: 2026-06-09 / Codex

- Decision: Move synthetic extension module classes from the CLI production scan root to `com.mycompany.seed4j.extension.runtime.main`, and use explicit fixture application start classes with `@Import`.
  Rationale: Keeping extension fixtures under `com.seed4j.cli` made the same-JVM Spring scan discover classes that were not installed in the active extension jar. External packages better model third-party extensions and prevent false positives from the test classpath.
  Date/Author: 2026-06-09 / Codex

## Risks and Mitigations

- Risk: Primary tests may accidentally duplicate production composition and drift from packaged startup.
  Mitigation: Compose with the same concrete adapters as `PreSpringBootstrapConfiguration` and keep packaged smoke tests for real jar execution.

- Risk: Moving the fixture can create broad import churn.
  Mitigation: Keep the fixture API small and update only tests that already use it.

- Risk: External packaged tests may require a built jar.
  Mitigation: Run `./mvnw -DskipTests package` before failsafe smoke validation.

## Validation Strategy

1. Run `./mvnw -Dtest=ExtensionRuntimeFixtureTest test` after moving the fixture.
2. Run `./mvnw -Dtest=ExtensionRuntimeBootstrapPrimaryTest test` during primary behavior cycles.
3. Run `./mvnw -Dtest='*Bootstrap*Test' test` after the primary suite replaces the domain journey suite.
4. Run `./mvnw -DskipTests package`, then packaged smoke failsafe tests.
5. Run `npm run prettier:check`.
6. Ask the user to run `./mvnw clean verify` locally and report the exit code and concise failure summary if any.

## Rollout and Recovery

This is a test-only restructuring. If validation fails after rollout, revert the test movement and restore the previous domain in-process test while investigating the failing primary observation point. No production runtime behavior should change.

## Lessons Learned

- Moving `ExtensionRuntimeFixture` to `bootstrap.fixture` required making only its fixture factory methods and result record public; the fixture regression stayed green.
- Same-JVM primary tests need to simulate both production scan isolation and production classpath/resource loading. Without that, broad tests can pass or fail for reasons caused by the test classpath rather than extension runtime behavior.
- Extension fixture classes under the CLI production scan root are a bad test shape for primary runner behavior because Spring can discover them from `target/test-classes`; placing them under a third-party package keeps the fixture honest.
- Same-JVM primary tests cannot faithfully prove extension overrides for Seed4J classpath resources that are resolved through parent-loaded dependency readers. Those override assertions remain in packaged jar coverage, while primary tests cover entrypoint behavior, module discovery, clean output, invalid jar failure, and extension module application.
