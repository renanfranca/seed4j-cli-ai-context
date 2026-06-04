# Fix Runtime Library Identity Logging

This ExecPlan is a living document. Keep `Progress`, `Decisions`, `Risks`, and `Lessons Learned` up to date as work advances.

## Purpose / Big Picture

Extension runtime library resolution should use Maven metadata when a nested library jar provides `pom.properties`, but debug logs should not claim a file-name override when the nested jar already has the expected Maven file name. Users running extension mode with debug logs should see override messages only for real mismatches, which keeps diagnostic output trustworthy.

## Scope

In scope: adjust `RuntimeExtensionLoaderPathResolver` logging behavior for Maven metadata identity, preserve existing loader path behavior, and add coverage for extension launch without `--debug` setting root logging to `ERROR`.

Out of scope: changing public domain APIs, moving runtime layout concepts into domain, changing command-line flags, or running `./mvnw clean verify` automatically.

Safety boundary: This task is limited to authorized, defensive maintenance of this repository. Do not provide offensive guidance or policy-bypassing instructions.

## Definitions

`RuntimeExtensionLoaderPathResolver` is a secondary infrastructure adapter that inspects the CLI executable jar and an extension jar to build Spring Boot `loader.path` entries for extension libraries missing from the CLI runtime.

`pom.properties` is Maven metadata inside a jar at `META-INF/maven/<groupId>/<artifactId>/pom.properties`, normally containing `groupId`, `artifactId`, and `version`.

Logical identity means the Maven coordinate and version used to decide if two runtime libraries are the same, for example `com.acme:shared-lib:1.0.0`.

Physical expected file name means the conventional Maven jar file name `<artifactId>-<version>.jar`, for example `shared-lib-1.0.0.jar`.

Override log means the debug message that says the resolver is using `pom.properties` identity instead of the identity inferred from a jar file name.

## Existing Context

`src/main/java/com/seed4j/cli/bootstrap/infrastructure/secondary/RuntimeExtensionLoaderPathResolver.java` currently reads nested `pom.properties`, maps it directly to `RuntimeLibraryIdentity`, compares it with `RuntimeLibraryIdentity.fromJarFileName(...)`, and logs an override whenever those identities differ. Because a file name only contains artifact and version, a normal Maven jar such as `shared-lib-1.0.0.jar` with metadata `com.acme:shared-lib:1.0.0` can look different if group information is included only in metadata.

`src/test/java/com/seed4j/cli/bootstrap/infrastructure/secondary/RuntimeExtensionLoaderPathResolverTest.java` already contains fixtures for jars with nested Maven metadata and a test for an actual mismatched file name.

`src/main/java/com/seed4j/cli/bootstrap/infrastructure/secondary/JavaProcessChildLauncher.java` already appears to set `logging.level.root=ERROR` for extension mode when `request.debug()` is false; a focused behavior test should lock that down.

## Desired End State

`RuntimeExtensionLoaderPathResolver` should parse Maven metadata into an infrastructure-private type that can expose both logical identity and expected Maven jar file name. The override debug log should be emitted only when metadata is present, the physical file name differs from `<artifactId>-<version>.jar`, and the identity inferred from the actual file name differs from the metadata identity. A standard Maven library with expected file name and complete metadata should not emit the override log.

`JavaProcessChildLauncherTest` should prove that extension launch without `--debug` produces a Java command containing `-Dlogging.level.root=ERROR`.

## Milestones

### Milestone 1 - Suppress False Override Log

#### Goal

Prove and fix the behavior where a normal Maven jar named `<artifactId>-<version>.jar` with group-aware `pom.properties` does not produce an override debug log.

#### Changes

- [ ] Edit `src/test/java/com/seed4j/cli/bootstrap/infrastructure/secondary/RuntimeExtensionLoaderPathResolverTest.java` to add a focused test for no override log on `shared-lib-1.0.0.jar` with metadata `com.acme:shared-lib:1.0.0`.
- [ ] Edit `src/main/java/com/seed4j/cli/bootstrap/infrastructure/secondary/RuntimeExtensionLoaderPathResolver.java` to introduce an infrastructure-private Maven metadata representation and log only when the physical file name is not the metadata expected file name.

#### Validation

- [ ] Command: `./mvnw -Dtest=RuntimeExtensionLoaderPathResolverTest test`
- [ ] Expected result: the resolver test class passes after first failing for the new no-log behavior.

#### Acceptance Criteria

- [ ] A normal Maven nested jar does not log the override message.
- [ ] Existing runtime library selection behavior remains unchanged.

### Milestone 2 - Preserve Real Override Log

#### Goal

Keep the existing real mismatch behavior: a file named `shared-lib-2.0.0.jar` with metadata `com.acme:shared-lib:1.0.0` should still log the metadata override.

#### Changes

- [ ] Keep or adapt the existing resolver test that expects the override message for mismatched physical file name and metadata identity.
- [ ] Ensure the production predicate distinguishes expected Maven names from incompatible file names.

#### Validation

- [ ] Command: `./mvnw -Dtest=RuntimeExtensionLoaderPathResolverTest test`
- [ ] Expected result: both no-log and real-override resolver behaviors pass.

#### Acceptance Criteria

- [ ] The override log appears for a real mismatched file name.
- [ ] The checkpoint resolver suite passes.

### Milestone 3 - Cover Non-Debug Extension Launcher Logging

#### Goal

Lock down extension child launch behavior without `--debug`, where root logging should be set to `ERROR`.

#### Changes

- [ ] Edit `src/test/java/com/seed4j/cli/bootstrap/infrastructure/secondary/JavaProcessChildLauncherTest.java` to add or adapt an extension-mode test without `--debug`.
- [ ] Production code changes are expected only if the test reveals a gap.

#### Validation

- [ ] Command: `./mvnw -Dtest=RuntimeExtensionLoaderPathResolverTest,JavaProcessChildLauncherTest test`
- [ ] Expected result: both focused test classes pass.

#### Acceptance Criteria

- [ ] The Java command for extension mode without debug contains `-Dlogging.level.root=ERROR`.
- [ ] The Java command for extension mode without debug does not contain the bootstrap domain debug logging property.

## Progress

- [x] ExecPlan created
- [x] Milestone 1 started
- [x] Milestone 1 completed
- [x] Milestone 2 started
- [x] Milestone 2 completed
- [x] Milestone 3 started
- [x] Milestone 3 completed
- [x] Focused final validation completed

## Decisions

- Decision: Keep Maven metadata modeling private to `RuntimeExtensionLoaderPathResolver`.
  Rationale: Maven metadata parsing and physical nested jar naming are secondary infrastructure concerns, and no public API needs to change.
  Date/Author: 2026-06-04 / Codex

- Decision: Emit the override log only when the actual nested jar file name differs from the Maven metadata expected file name and the file-name identity is different from metadata identity.
  Rationale: This keeps logs quiet for normal Maven jars while preserving diagnostics for renamed or inconsistent nested libraries.
  Date/Author: 2026-06-04 / Codex

- Decision: Add launcher coverage without changing `JavaProcessChildLauncher`.
  Rationale: The requested non-debug extension behavior was already implemented; the missing piece was an observable command-building test.
  Date/Author: 2026-06-04 / Codex

## Risks and Mitigations

- Risk: Comparing the metadata expected physical file name could accidentally suppress logs for truly renamed libraries.
  Mitigation: Keep a dedicated mismatch test where the actual jar name differs from `<artifactId>-<version>.jar`.

- Risk: Jar fixture helpers can make expected values depend on production logic.
  Mitigation: Keep assertions explicit in test bodies and use helper methods only for jar construction.

## Validation Strategy

1. Run `./mvnw -Dtest=RuntimeExtensionLoaderPathResolverTest test` for resolver behavior after each resolver cycle.
2. Run `./mvnw -Dtest=RuntimeExtensionLoaderPathResolverTest,JavaProcessChildLauncherTest test` after launcher coverage is added.
3. Do not run `./mvnw clean verify` automatically; ask the user to run it locally if a final repository gate is needed.

## Rollout and Recovery

This is an internal diagnostic logging and test coverage change. Rollout is through the normal CLI build and release process. Recovery is to revert the changes in `RuntimeExtensionLoaderPathResolver.java` and the corresponding tests if unexpected logging or runtime library selection regressions appear.

## Lessons Learned

- `./mvnw -Dtest=RuntimeExtensionLoaderPathResolverTest test` runs the Maven enforcer and shows existing dependency convergence warnings, but the focused test goal can still complete successfully.
- The previous resolver predicate compared group-aware metadata identity with group-less file-name inference, which explains the false positive debug log for normal Maven jar names.
- `JavaProcessChildLauncher` already set `logging.level.root=ERROR` for extension mode without debug; the new test locks down that existing behavior.
