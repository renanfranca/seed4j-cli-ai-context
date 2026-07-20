# Add opt-in PIT mutation testing

This ExecPlan is a living document. Keep `Progress`, `Decisions`, `Risks`, and `Lessons Learned` up to date as work advances.

## Purpose / Big Picture

Maintainers need an explicit mutation-testing command that measures whether the existing tests detect small behavioral changes. Activating the Maven `pitest` profile will run PIT against Seed4J CLI production classes and unit/component tests, write an HTML report under `target/pit-reports/`, and leave the normal Maven lifecycle unchanged.

## Scope

In scope: fixed PIT Maven and JUnit Platform plugin versions, an opt-in Maven profile scoped to `com.seed4j.cli.*`, exclusion of packaged-JAR integration and Cucumber tests, README usage guidance, a diagnostic dry run, a full baseline run, and normal test/format validation.

Out of scope: mutation thresholds, lifecycle bindings, CI workflow changes, scheduled runs, extra mutators, incremental analysis, and tests written solely to improve the initial mutation score.

## Definitions

PIT is a mutation-testing tool that changes compiled bytecode in small ways and checks whether tests fail. A killed mutant is detected by a test; a surviving mutant identifies behavior that the current tests may not distinguish. An opt-in Maven profile is inactive unless selected with `-Ppitest`. A dry run discovers tests and generates mutants without executing each test suite against every mutant.

## Existing Context

The single-module Maven project is configured in `pom.xml`. Surefire excludes `**/*IT*` and `**/*CucumberTest*`; Failsafe owns those packaged-JAR and Cucumber scenarios. The default GitHub workflow invokes `clean verify`. `README.md` documents local source builds near the beginning of the file. The project uses Spring Boot 4.1 and JUnit Platform 6, so PIT requires its JUnit 5/Platform test plugin and must be validated against this combination.

## Desired End State

`./mvnw -Ppitest test-compile org.pitest:pitest-maven:mutationCoverage` discovers Seed4J CLI tests, mutates Seed4J CLI production classes, excludes integration/Cucumber tests, and produces a non-empty report. Running Maven without `-Ppitest` neither resolves nor executes PIT as part of the lifecycle. The README explains the command, cost, report location, and interpretation of survivors.

## Milestones

### Milestone 1 - Configure the opt-in profile

#### Goal

Introduce the smallest Maven configuration that can discover and mutate the intended classes and tests.

#### Changes

- [ ] Add fixed `pitest-maven` and `pitest-junit5-plugin` version properties to `pom.xml`.
- [ ] Add an inactive-by-default `pitest` profile with package filters and test exclusions matching Surefire.
- [ ] Leave the default build and GitHub Actions workflow unchanged.

#### Validation

- [ ] Command: `./mvnw -Ppitest -Dpit.dryRun=true test-compile org.pitest:pitest-maven:mutationCoverage`
- [ ] Expected result: PIT discovers tests and production classes, generates mutants, and writes a report with non-zero test/coverage data.

#### Acceptance Criteria

- [ ] The profile is activated only by `-Ppitest`.
- [ ] `*IT*` and `*CucumberTest*` are not selected by PIT.

### Milestone 2 - Document maintainer usage

#### Goal

Make the opt-in workflow and its output unambiguous to maintainers.

#### Changes

- [ ] Add a mutation-testing subsection near the local build instructions in `README.md`.
- [ ] Explain the explicit command, report location, execution cost, and survivor meaning.

#### Validation

- [ ] Command: `npm run prettier:check`
- [ ] Expected result: Maven XML and Markdown formatting conform to the repository formatter.

#### Acceptance Criteria

- [ ] A maintainer can run and interpret PIT using only the README.

### Milestone 3 - Establish and verify the baseline

#### Goal

Prove that the integration works with the real suite and record the initial result without enforcing it.

#### Changes

- [x] Run full mutation coverage and record duration, mutant count, and score here.
- [x] Run the normal unit test suite and formatting check.
- [x] Inspect the effective default lifecycle/profile state to confirm PIT remains opt-in.

#### Validation

- [ ] Command: `./mvnw -Ppitest test-compile org.pitest:pitest-maven:mutationCoverage`
- [ ] Command: `./mvnw test`
- [ ] Command: `npm run prettier:check`
- [ ] Expected result: all commands succeed; PIT reports non-zero mutants and tests; no threshold is introduced.

#### Acceptance Criteria

- [ ] The baseline is measurable and the ordinary test suite remains green.
- [ ] The user is asked to run `./mvnw clean verify` locally because repository policy reserves the complete gate for the user unless explicitly requested.

## Progress

- [x] ExecPlan created and repository context inspected.
- [x] Milestone 1 started.
- [x] Milestone 1 completed.
- [x] Milestone 2 started.
- [x] Milestone 2 completed.
- [x] Milestone 3 started.
- [x] Milestone 3 completed.

## Decisions

- Decision: Keep mutation testing entirely inside an inactive Maven profile and invoke its goal explicitly.
  Rationale: The existing suite is large enough that multiplying test executions would make normal builds and CI materially slower.
  Date/Author: 2026-07-20 / Codex
- Decision: Mirror Surefire's `*IT*` and `*CucumberTest*` exclusions in PIT.
  Rationale: These tests belong to Failsafe and depend on packaged application artifacts, so they are unsuitable for the `test-compile` mutation path.
  Date/Author: 2026-07-20 / Codex
- Decision: Pin `pitest-maven` 1.25.8 and `pitest-junit5-plugin` 1.2.3.
  Rationale: Maven Central metadata on 2026-07-20 identifies these as the latest releases, and fixed versions keep the opt-in analysis reproducible.
  Date/Author: 2026-07-20 / Codex

## Risks and Mitigations

- Risk: PIT's JUnit plugin may not interoperate with JUnit Platform 6 used through Spring Boot 4.1.
  Mitigation: Run PIT dry-run discovery first and require non-zero discovered tests and coverage before the full mutation run.
- Risk: Mutation testing can be slow or resource intensive.
  Mitigation: Keep it opt-in, avoid lifecycle/CI binding, and document the cost.
- Risk: A low initial score could encourage fragile implementation-detail tests.
  Mitigation: Record the baseline without a threshold and interpret survivors as investigation prompts rather than an automatic mandate.

## Validation Strategy

1. Use PIT dry-run mode as the first behavioral checkpoint.
2. Format `pom.xml` and `README.md`, then check formatting.
3. Run full mutation coverage and capture its summary and elapsed time.
4. Run `./mvnw test` as the repository's default agent-side gate.
5. Inspect Maven profile/default lifecycle configuration without running the prohibited automatic `clean verify` gate.
6. Ask the user to run `./mvnw clean verify` locally and return its exit code and any relevant failure summary.

## Rollout and Recovery

No deployment or data migration is involved. The profile becomes available when the POM lands and affects only callers that pass `-Ppitest`. Recovery is a normal revert of the POM and README additions; generated reports under `target/` are disposable build output.

## Lessons Learned

- Maven Central must be checked at implementation time because the requested fixed versions are time-sensitive.
- Before the profile existed, the diagnostic command selected PIT 1.25.8 dynamically, warned that `pitest-junit5-plugin` was missing, sent 294 test classes to the minion, and failed with zero runnable tests. This is the expected RED evidence for the profile's JUnit integration behavior.
- With the profile enabled, dry-run discovery succeeded with JUnit Platform 6.0.3: PIT examined 277 tests, generated 1,025 mutations, measured 2,212/2,249 covered lines (98%), and reported zero uncovered mutations. Dry-run mode intentionally ran zero tests against mutants.
- The full mutation baseline completed successfully in 21 minutes and 46 seconds. PIT examined 277 tests, generated 1,025 mutations, killed 861 (84%), reported 8 without coverage, measured 85% test strength, and ran 3,913 tests against mutants.
- The normal Maven suite passed all 484 tests in 22.186 seconds. The changed POM, README, and ExecPlan pass Prettier; the repository-wide Prettier check still reports eight unrelated, pre-existing files.
