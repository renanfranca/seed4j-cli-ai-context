# Refactor runtime extension installation tests

This ExecPlan is a living document. Keep `Progress`, `Decisions`, `Risks`, and `Lessons Learned` up to date as work advances.

## Purpose / Big Picture

Refactor runtime extension installation tests so they observe behavior through stable public contracts instead of internal orchestration details. A maintainer should be able to trust that installation works through `JavaRuntimeExtensionInstaller.install(...)` and that filesystem artifact publication is covered through the secondary repository's observable filesystem effects.

## Scope

In scope: replace `RuntimeExtensionInstallerTest` with behavior tests in `bootstrap/infrastructure/primary` and `bootstrap/infrastructure/secondary`, preserving production behavior. Out of scope: changing runtime installation semantics, changing CLI command behavior, changing runtime configuration file format, or running the full `./mvnw clean verify` gate automatically.

Safety boundary: This task is limited to authorized, defensive maintenance of this repository. Do not provide offensive guidance or policy-bypassing instructions.

## Definitions

`JavaRuntimeExtensionInstaller` is the primary Java API used by callers to install an extension runtime. `FileSystemRuntimeExtensionArtifactsRepository` is the secondary adapter that publishes the active extension jar and metadata files. `Runtime artifacts` means the active runtime jar and metadata under the CLI home runtime directory. `Config` means `~/.config/seed4j-cli/config.yml` under a test-specific temporary home.

## Existing Context

`src/test/java/com/seed4j/cli/bootstrap/domain/RuntimeExtensionInstallerTest.java` currently mixes behavior tests with fake repositories that assert call counts and call order. It also covers real filesystem publication by constructing the domain installer with concrete secondary repositories. `src/main/java/com/seed4j/cli/bootstrap/infrastructure/primary/JavaRuntimeExtensionInstaller.java` wraps domain installation failures as `JavaRuntimeExtensionInstallationException`. `src/main/java/com/seed4j/cli/bootstrap/infrastructure/secondary/FileSystemRuntimeExtensionArtifactsRepository.java` publishes jar and metadata files using temporary files and detects an active runtime when either artifact exists.

## Desired End State

There is no `RuntimeExtensionInstallerTest` in the domain package. Installation behavior with real validators/repositories is covered by `JavaRuntimeExtensionInstallerTest` through the public primary API. Filesystem publication cleanup and active-runtime detection are covered by `FileSystemRuntimeExtensionArtifactsRepositoryTest` through observable filesystem state. Tests avoid assertions about fake call order, fake counters, or temporary filename implementation details.

## Milestones

### Milestone 1 - Primary installation behavior

#### Goal

Cover successful and failed extension installation through `JavaRuntimeExtensionInstaller.install(...)`.

#### Changes

- [ ] Add `src/test/java/com/seed4j/cli/bootstrap/infrastructure/primary/JavaRuntimeExtensionInstallerTest.java`.
- [ ] Construct the primary with `RuntimeExtensionApplicationService` and real secondary/domain collaborators using a temporary CLI home.
- [ ] Cover config creation, artifact copy, metadata writing, config key preservation, runtime replacement, invalid config wrapping, and invalid jar wrapping.

#### Validation

- [ ] Command: `./mvnw -Dtest=JavaRuntimeExtensionInstallerTest test`
- [ ] Expected result: new primary tests pass.

#### Acceptance Criteria

- [ ] Failures are observed as `JavaRuntimeExtensionInstallationException`.
- [ ] Tests call `JavaRuntimeExtensionInstaller.install(...)`, not the domain installer directly.

### Milestone 2 - Secondary filesystem behavior

#### Goal

Cover publication cleanup and active runtime detection through filesystem effects.

#### Changes

- [ ] Add `src/test/java/com/seed4j/cli/bootstrap/infrastructure/secondary/FileSystemRuntimeExtensionArtifactsRepositoryTest.java`.
- [ ] Cover failed jar publication without unexpected leftovers.
- [ ] Cover failed metadata publication without unexpected leftovers while preserving current behavior where the jar may already be published.
- [ ] Cover `activeRuntimePresent()` when only jar exists and when only metadata exists.

#### Validation

- [ ] Command: `./mvnw -Dtest=FileSystemRuntimeExtensionArtifactsRepositoryTest test`
- [ ] Expected result: new secondary tests pass.

#### Acceptance Criteria

- [ ] Tests do not assert internal temporary filename prefixes.
- [ ] Tests assert observable directory contents and published artifact state.

### Milestone 3 - Remove old orchestration test and checkpoint

#### Goal

Remove obsolete domain test coverage and verify nearby public paths remain green.

#### Changes

- [ ] Delete `src/test/java/com/seed4j/cli/bootstrap/domain/RuntimeExtensionInstallerTest.java`.
- [ ] Run targeted test commands from the user plan.
- [ ] Run formatter check.

#### Validation

- [ ] Command: `./mvnw -Dtest=JavaRuntimeExtensionInstallerTest,FileSystemRuntimeExtensionArtifactsRepositoryTest test`
- [ ] Command: `./mvnw -Dtest=ExtensionInstallCommandTest,ExtensionRuntimeBootstrapPrimaryTest test`
- [ ] Command: `npm run prettier:check`
- [ ] Expected result: all focused checks pass.

#### Acceptance Criteria

- [ ] Domain fake-based orchestration assertions are gone.
- [ ] Focused checks pass or failures are reported with evidence.

## Progress

- [x] ExecPlan created
- [x] Milestone 1 started
- [x] Milestone 1 completed
- [x] Milestone 2 started
- [x] Milestone 2 completed
- [x] Milestone 3 started
- [x] Milestone 3 completed

## Decisions

- Decision: Keep this task test-only.
  Rationale: The user plan explicitly says not to alter production behavior.
  Date/Author: 2026-06-09 / Codex
- Decision: Use `@UnitTest` for the new filesystem-backed tests.
  Rationale: The user plan records this assumption and the tests remain focused without Spring context startup.
  Date/Author: 2026-06-09 / Codex

## Risks and Mitigations

- Risk: New tests could mirror production topology instead of behavior.
  Mitigation: Use the primary public API for installation behavior and the secondary adapter only for its stable filesystem contract.
- Risk: Filesystem failure assertions could depend on private temporary filenames.
  Mitigation: Assert directory contents and absence of unexpected leftovers rather than temporary filename prefixes.
- Risk: Wrapping exceptions at the primary could hide useful cause checks.
  Mitigation: Assert the primary exception type and cause type/message where behavior requires it.

## Validation Strategy

1. Run focused tests for the new primary and secondary behavior.
2. Run existing command/bootstrap public-path tests listed by the user.
3. Run `npm run prettier:check`.
4. Ask the user to run `./mvnw clean verify` locally if a final full gate is needed.

## Rollout and Recovery

This is a test refactor. Rollout is merging the test changes after focused checks pass. Recovery is reverting the test files added and restoring the deleted domain test if the new observation points miss behavior.

## Lessons Learned

- `JavaRuntimeExtensionInstallerTest` passes through the primary API with the real validator, mode repository, and artifacts repository.
- `FileSystemRuntimeExtensionArtifactsRepositoryTest` passes while asserting visible directory contents instead of temporary filename prefixes.
- Focused checks passed:
  `./mvnw -Dtest=JavaRuntimeExtensionInstallerTest,FileSystemRuntimeExtensionArtifactsRepositoryTest test`,
  `./mvnw -Dtest=ExtensionInstallCommandTest,ExtensionRuntimeBootstrapPrimaryTest test`, and
  `npm run prettier:check`.
