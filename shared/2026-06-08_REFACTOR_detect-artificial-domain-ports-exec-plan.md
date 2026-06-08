# Detect Artificial Domain Ports

This ExecPlan is a living document. Keep `Progress`, `Decisions`, `Risks`, and `Lessons Learned` up to date as work advances.

## Purpose / Big Picture

The repository should fail architecture tests when a domain interface is only a label for secondary infrastructure internals. This makes the hexagonal boundary more observable: a real domain port must be needed by domain or application behavior, while adapter-internal seams stay in infrastructure.

## Scope

In scope: add an ArchUnit rule in `HexagonalArchTest`, remove artificial bootstrap domain interfaces, add an infrastructure seam for child process execution, update affected tests and documentation, and run focused validation.

Out of scope: changing package conventions, banning all `Path` values in domain, or running `./mvnw clean verify` automatically.

## Definitions

A domain port is an interface in a `..domain..` package that models a capability needed by domain or application behavior. A secondary adapter is a class in `..infrastructure.secondary..` that talks to technical mechanisms such as files, processes, or runtime metadata. An artificial domain port is a domain interface implemented by secondary infrastructure but not consumed by production domain or application code.

## Existing Context

`src/test/java/com/seed4j/cli/HexagonalArchTest.java` already checks domain naming and dependency boundaries. Bootstrap secondary classes currently implement six domain interfaces that are not used by domain or application behavior: `PreSpringRuntimeEnvironmentReader`, `RuntimeExtensionCacheIdentityResolver`, `RuntimeExtensionLoaderPathResolver`, `RuntimeExtensionOverlayCache`, `RuntimeExtensionStartClassResolver`, and `ProcessCommandExecutor`. `PreSpringBootstrapConfiguration` wires concrete secondary classes directly.

## Desired End State

`HexagonalArchTest` fails when a `..domain..` interface implemented by `..infrastructure.secondary..` has no production dependency from `..domain..` or `..application..`. The six artificial bootstrap interfaces are removed. `JavaProcessChildLauncher` still has a test seam through an infrastructure interface named `ChildProcessCommandExecutor`. Documentation explains the rule.

## Milestones

### Milestone 1 - Architecture Rule

#### Goal

Make secondary-only domain interfaces visible as an architecture violation.

#### Changes

- [ ] Edit `src/test/java/com/seed4j/cli/HexagonalArchTest.java` to add a custom condition under `Secondary`.

#### Validation

- [ ] Command: `./mvnw -Dtest=HexagonalArchTest test`
- [ ] Expected result: fails for the six artificial bootstrap interfaces before cleanup.

#### Acceptance Criteria

- [ ] The failure names each artificial interface.
- [ ] Secondary, composition, and tests do not count as valid consumers.

### Milestone 2 - Remove Artificial Ports

#### Goal

Keep the same runtime behavior while removing incorrect domain API.

#### Changes

- [ ] Delete the six artificial interfaces under `src/main/java/com/seed4j/cli/bootstrap/domain`.
- [ ] Remove `implements` clauses from the secondary resolver/cache/environment classes.
- [ ] Add `src/main/java/com/seed4j/cli/bootstrap/infrastructure/secondary/ChildProcessCommandExecutor.java`.
- [ ] Update `JavaChildProcessCommandExecutor`, `JavaProcessChildLauncher`, and affected tests to use the infrastructure seam.

#### Validation

- [ ] Command: `./mvnw -Dtest=HexagonalArchTest test`
- [ ] Expected result: passes after cleanup.

#### Acceptance Criteria

- [ ] Production code compiles.
- [ ] The new rule has no remaining violations.

### Milestone 3 - Focused Behavior and Documentation

#### Goal

Confirm bootstrap behavior is unchanged and document the architecture expectation.

#### Changes

- [ ] Edit `documentation/hexagonal-architecture.md` to explain that domain ports must be consumed by domain/application behavior.

#### Validation

- [ ] Command: `./mvnw -Dtest=JavaProcessChildLauncherTest,JavaChildProcessCommandExecutorTest,CurrentProcessPreSpringRuntimeEnvironmentReaderTest,RuntimeExtensionCacheIdentityResolverTest,RuntimeExtensionLoaderPathResolverTest,RuntimeExtensionOverlayCacheTest,RuntimeExtensionStartClassResolverTest test`
- [ ] Expected result: passes.
- [ ] Command: `./mvnw test`
- [ ] Expected result: passes.
- [ ] Command: `npm run prettier:check`
- [ ] Expected result: passes.

#### Acceptance Criteria

- [ ] Focused bootstrap tests pass.
- [ ] Repository unit tests pass.
- [ ] Formatting check passes.

## Progress

- [x] ExecPlan created
- [x] Milestone 1 started
- [x] Milestone 1 completed
- [x] Milestone 2 started
- [x] Milestone 2 completed
- [x] Milestone 3 started
- [x] Milestone 3 completed

## Decisions

- Decision: Use an ArchUnit custom condition instead of a naming convention.
  Rationale: The target behavior depends on implementation and consumer relationships, not only package or name.
  Date/Author: 2026-06-08 / Codex
- Decision: Keep process execution as `ChildProcessCommandExecutor` in secondary infrastructure.
  Rationale: `JavaProcessChildLauncher` still benefits from a deterministic process-execution seam, but that seam is technical and not consumed by domain or application behavior.
  Date/Author: 2026-06-08 / Codex

## Risks and Mitigations

- Risk: ArchUnit dependency traversal could count implementation inheritance from secondary as a valid consumer.
  Mitigation: Only accept dependencies whose origin owner resides in `..domain..` or `..application..`.
- Risk: Removing interfaces could accidentally weaken a useful test seam.
  Mitigation: Keep the process-launch seam as an infrastructure interface used by `JavaProcessChildLauncherTest`.

## Validation Strategy

1. Run `./mvnw -Dtest=HexagonalArchTest test` red, then green.
2. Run focused bootstrap secondary tests.
3. Run `./mvnw test` and `npm run prettier:check`.
4. Ask the user to run `./mvnw clean verify` locally if they need the final gate.

## Rollout and Recovery

This is an internal architecture and refactor change. If it causes release trouble, revert the commit containing the rule and interface cleanup together, because the rule intentionally protects the cleanup from regressing.

## Lessons Learned

- The new ArchUnit rule failed RED exactly on the six bootstrap interfaces from the plan: `PreSpringRuntimeEnvironmentReader`, `ProcessCommandExecutor`, `RuntimeExtensionCacheIdentityResolver`, `RuntimeExtensionLoaderPathResolver`, `RuntimeExtensionOverlayCache`, and `RuntimeExtensionStartClassResolver`.
- `./mvnw -Dtest=HexagonalArchTest test` passes after cleanup.
- Focused bootstrap secondary tests passed: 47 tests, 0 failures.
- `./mvnw test` passed: 502 tests, 0 failures.
- `npm run prettier:check` initially found formatting drift in `HexagonalArchTest.java`; after running Prettier on touched files, the check passed.
