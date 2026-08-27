# Resolve confirmed quality risks and Sonar follow-up

This ExecPlan is a living document. Keep `Progress`, `Decisions`, `Risks`, and `Lessons Learned` up to date as work advances.

## Purpose / Big Picture

This authorized maintenance removes seven design risks confirmed by Habit Hooks and then closes the Sonar issues exposed by the pull-request gate, while preserving every public Java API, CLI option, message, exit code, persisted file, and user-visible workflow. The result is observable through the unchanged CLI behavior, with one regression scenario proving that independently built command trees no longer share Bash-completion invocation state and a final local Sonar analysis reporting zero overall code smells.

## Scope

In scope are the Bash completion command lifecycle, Bash completion candidate collection and script rendering, child JVM request construction, lazy CLI version rendering, atomic publication of bootstrap files, shared Picocli metadata for the `--project-path` option, and the eleven Sonar findings on record-pattern destructuring and a method-reference simplification. The original seven changes remain independently committed; the Sonar follow-up will be one focused commit that updates the active pull request.

Out of scope are changes to public Java APIs, CLI syntax, messages, exit codes, persistence formats, user documentation, `BusinessContext` and `SharedKernel`, cross-context runtime DTO consolidation, generic test-fixture cleanup, assertion deduplication, and tests that exist only to protect wiring, annotations, delegation, record-pattern syntax, or internal topology. The complete `./mvnw clean verify` gate remains a user-run checkpoint unless the user later explicitly authorizes the agent to run it.

## Definitions

An invocation is one parsed execution of a Picocli command tree. A `CommandSpec` is Picocli's technical model of commands and options. Value completion is the Bash script behavior that suggests values supplied by an option's completion candidates. A child process request is an immutable technical snapshot containing the JVM launch facts resolved before command rendering. Atomic publication means writing or copying to an adjacent temporary file and then moving it into the target location atomically when supported, with a replacement fallback. A public-path checkpoint exercises behavior through the existing CLI, bootstrap primary adapter, or persistence-facing test rather than a newly extracted helper. A Java record pattern destructures a record directly in a `switch` case, replacing a type pattern followed by calls to that record's component accessors. Sonar rule `java:S6878` requests that destructuring, while `java:S1612` requests an equivalent method reference in place of a forwarding lambda.

## Existing Context

`BashCompletionCommand` is a Spring-managed reusable command object whose `spec()` method replaces a mutable `commandSpec` field. Building a second root tree can therefore make execution of the first tree read the second tree's option state. `BashCompletionScriptGenerator` currently owns both recursive Picocli metadata traversal and the complete Bash template/rendering algorithm.

`JavaProcessChildLauncher` currently resolves runtime extension metadata and overlay/loader details while also ordering JVM arguments and executing the process. Pre-Spring composition constructs this secondary adapter explicitly. `Seed4JCommandsFactory` currently resolves runtime display while building every command tree, even when `--version` is never requested.

`RuntimeModeConfigurationWriter` and `FileSystemRuntimeExtensionArtifactsRepository` independently implement adjacent temporary-file publication with atomic move fallback and cleanup. `ApplyModuleCommand` and `ApplyModuleSetCommand` independently spell the same Picocli metadata for `--project-path`, although their domain values and application flows are intentionally distinct.

Canonical architecture guidance is in `AGENTS.md` and `documentation/`. These sources require explicit hexagonal boundaries, types-driven business modeling, interface syntax at the primary boundary, operational paths at the secondary boundary, and explicit pre-Spring composition.

## Desired End State

Every `BashCompletionCommand.spec()` call returns a command backed by a new short-lived invocation object that owns its `CommandSpec` and parsed options. A package-private collector returns immutable, ordered Bash completion candidate maps, a package-private renderer owns all Bash text and quoting, and the generator only coordinates them.

A package-private `JavaChildProcessRequestFactory` resolves the technical runtime snapshot represented by a package-private immutable request; `JavaProcessChildLauncher` only renders the ordered JVM command and executes it. Pre-Spring composition wires the new technical collaborator without adding a domain port.

A package-private Spring-managed `Seed4JVersionProvider` implements Picocli `IVersionProvider`; it reads runtime display and renders the exact existing standard, extension, and fallback output only when Picocli requests version information. The root command factory depends only on registered commands and this provider.

A package-private `AtomicFilePublisher` owns adjacent temporary creation, atomic move, replacement fallback, cleanup, and exception propagation for both content and source publication. A package-private `ProjectPathOptionSpecFactory` produces a fresh `OptionSpec` for each command while sharing only the existing `--project-path` presentation metadata.

Every Sonar `java:S6878` match introduced on this branch uses a Java 25 record pattern with the smallest component binding needed by the existing branch, and the single `java:S1612` forwarding lambda uses its equivalent method reference. The local Sonar overall metric, not only the new-code metric, reports zero code smells for the branch head.

## Milestones

### Milestone 1 - Isolate Bash completion invocation state

#### Goal

Prove and fix command-tree cross-contamination when two roots are built from the same Spring command objects.

#### Changes

Add one behavior test to `src/test/java/com/seed4j/cli/command/infrastructure/primary/Seed4JCommandsFactoryTest.java` that builds two roots, sets `--no-complete-values` only on the first, and executes the first. Confirm the test fails because the first tree reads state from the second. Refactor `src/main/java/com/seed4j/cli/command/infrastructure/primary/BashCompletionCommand.java` so each `spec()` creates an invocation owning its own options and `CommandSpec` while preserving generation, installation, output, and exit-code behavior. No canonical documentation changes are needed because public behavior is restored rather than changed.

#### Validation

Run `./mvnw -Dtest=BashCompletionScriptGeneratorTest,Seed4JCommandsFactoryTest test`. RED must fail for the new two-tree scenario for the expected stale-state reason; GREEN must pass the complete focused suites. Run Habit Hooks on changed files and analyze every finding. Commit as `fix(command): isolate bash completion invocation state` with the requested explanatory body.

#### Acceptance Criteria

The first tree omits value-completion code when invoked with `completion bash --no-complete-values` even after a second tree is built, while all existing Bash generation/installation tests remain green.

### Milestone 2 - Extract the child process request factory

#### Goal

Separate runtime fact resolution from JVM command rendering and process execution without changing the pre-Spring boundary.

#### Changes

Add package-private `JavaChildProcessRequestFactory` and an immutable package-private technical request under `src/main/java/com/seed4j/cli/bootstrap/infrastructure/secondary/`. Move start-class, overlay, loader-path, and extension-property resolution into the factory. Reduce `JavaProcessChildLauncher` to ordered command rendering and execution. Preserve the public launcher constructor used by the pre-Spring composition; because Java package access prevents that separate composition package from naming the new package-private factory, construct the internal factory at the launcher's construction boundary from the same explicitly supplied secondary collaborators. Do not add a new domain port. Continue validating through existing launcher and bootstrap-primary behavior tests; do not add topology tests. No documentation change is required because the established secondary/composition guidance remains accurate.

#### Validation

Run `./mvnw -Dtest=JavaProcessChildLauncherTest,PreSpringBootstrapPrimaryTest test`, then Habit Hooks on changed files. Commit as `refactor(bootstrap): extract child process request factory` with the requested explanatory body.

#### Acceptance Criteria

Existing tests observe identical JVM argument order, properties, overlay/loader behavior, process execution, errors, and primary bootstrap results.

### Milestone 3 - Extract the lazy CLI version provider

#### Goal

Keep command-tree construction free of runtime-display reads and render the exact established version output only when Picocli requests it.

#### Changes

Add package-private Spring-managed `Seed4JVersionProvider` under `src/main/java/com/seed4j/cli/command/infrastructure/primary/`, implementing `CommandLine.IVersionProvider`. Move project-version fallback, runtime-display lookup, and standard/extension rendering into it. Change `Seed4JCommandsFactory` to depend only on the command list and provider and attach that provider to the root `CommandSpec`. Add or adjust behavior assertions only in the existing `Seed4JCommandsFactoryTest` where needed to prove the lazy public behavior; do not test annotations or wiring. No user documentation changes are needed because `--version` output stays byte-for-byte compatible.

#### Validation

Run `./mvnw -Dtest=Seed4JCommandsFactoryTest test`, then Habit Hooks on changed files. Commit as `refactor(command): extract CLI version provider` with the requested explanatory body.

#### Acceptance Criteria

All standard, extension, blank-version, and fallback version outputs remain identical, and building or executing unrelated commands does not query runtime display.

### Milestone 4 - Separate Bash completion candidate collection

#### Goal

Give recursive Picocli traversal and ordered candidate ownership one cohesive technical component.

#### Changes

Add package-private `BashCompletionCandidateCollector` and package-private immutable `BashCompletionCandidates` under the command primary adapter package. Move traversal of commands, options, subcommands, negated option names, value options, and completion candidates out of the generator. Defensively own immutable, sorted maps and immutable candidate lists while preserving duplicate and ordering semantics. Keep existing public-path tests in `BashCompletionScriptGeneratorTest` and `Seed4JCommandsFactoryTest`; do not add tests for the extracted topology. No documentation change is required because the generated script contract does not change.

#### Validation

Run `./mvnw -Dtest=BashCompletionScriptGeneratorTest,Seed4JCommandsFactoryTest test`, then Habit Hooks on changed files. Commit as `refactor(command): separate bash completion candidate collection` with the requested explanatory body.

#### Acceptance Criteria

The complete generated script remains unchanged for current command trees and every candidate map is immutable and deterministically ordered internally.

### Milestone 5 - Extract the Bash completion script renderer

#### Goal

Separate Bash text generation from candidate discovery so the generator becomes a small coordinator.

#### Changes

Add package-private `BashCompletionScriptRenderer` under the command primary adapter package. Move the Bash template, path and value `case` blocks, shell quoting, and separated/attached value-completion algorithm into it. Reduce `BashCompletionScriptGenerator` to collector/renderer coordination. Continue protecting behavior only through the existing generated-script tests and CLI path. No documentation change is required because script bytes and shell behavior remain stable.

#### Validation

Run `./mvnw -Dtest=BashCompletionScriptGeneratorTest,Seed4JCommandsFactoryTest test`, then Habit Hooks on changed files. Commit as `refactor(command): extract bash completion script renderer` with the requested explanatory body.

#### Acceptance Criteria

Existing script snapshots/fragments, quoting cases, option-value algorithms, and CLI output remain unchanged.

### Milestone 6 - Centralize atomic file publication

#### Goal

Make the two bootstrap persistence paths share one complete and consistent atomic-publication policy.

#### Changes

Add package-private `AtomicFilePublisher` under `src/main/java/com/seed4j/cli/bootstrap/infrastructure/secondary/` with `publishContent(String content, Path targetPath)` and `publishSource(Path sourcePath, Path targetPath)`. Centralize adjacent temporary-file creation, `ATOMIC_MOVE`, fallback with `REPLACE_EXISTING`, temporary cleanup on failure, and propagation of the original publication exception. Reuse it from `RuntimeModeConfigurationWriter` and `FileSystemRuntimeExtensionArtifactsRepository`, preserving their public constructors/ports and persisted bytes. Continue testing via the existing writer and repository behavior tests rather than the new internal class. No user documentation changes are needed because paths, file names, formats, and failure contracts do not change.

#### Validation

Run `./mvnw -Dtest=RuntimeModeConfigurationWriterTest,FileSystemRuntimeExtensionArtifactsRepositoryTest test`, then Habit Hooks on changed files. Commit as `refactor(bootstrap): centralize atomic file publication` with the requested explanatory body.

#### Acceptance Criteria

Both callers publish the same bytes to the same targets, replace existing targets as before, leave no adjacent temporary artifacts after failures, and propagate the established exceptions.

### Milestone 7 - Share project-path option metadata

#### Goal

Remove duplicated Picocli presentation metadata while keeping apply and apply-set domain inputs and application flows separate.

#### Changes

Add package-private `ProjectPathOptionSpecFactory` under the command primary adapter package. Make it return a new `OptionSpec` per call containing only the current spelling, description, parameter label, default, and completion provider for `--project-path`. Reuse it from `ApplyModuleSubCommand` and `ApplyModuleSetCommand` without merging their domain types, parse values, or execution paths. Validate through existing public command/application tests; do not add internal metadata-factory tests. No documentation change is needed because help, defaults, completion, and option behavior remain identical.

#### Validation

Run focused public command tests covering apply and apply-set after identifying their existing test homes, then Habit Hooks on changed files. Commit as `refactor(command): share project path option metadata` with the requested explanatory body.

#### Acceptance Criteria

Both command families render and parse the same established `--project-path` metadata, each specification receives an independent mutable Picocli option model, and their domain/application workflows remain separate.

### Milestone 8 - Close Sonar record-pattern findings

#### Goal

Remove all eleven code smells reported by the local and pull-request Sonar analyses without changing module-set planning, rendering, history mapping, CLI output, or public contracts.

#### Changes

Use record patterns in `src/main/java/com/seed4j/cli/command/domain/moduleset/ModuleSetParameterResolutionSummary.java`, `src/main/java/com/seed4j/cli/command/infrastructure/primary/ApplyModuleSetPlanRenderer.java`, `src/main/java/com/seed4j/cli/command/infrastructure/primary/ApplyModuleSetPlanningProblemRenderer.java`, and `src/main/java/com/seed4j/cli/command/infrastructure/secondary/ProjectsModuleSetPlanningHistoryReader.java`. Replace the one forwarding lambda in the planning-problem renderer with `ModuleSetPropertyDefaultValue::literal`. Do not add tests for syntax or internal topology. `README.md` and `documentation/` remain unchanged because the refactor preserves all user-visible behavior.

#### Validation

Use the current local Sonar analysis on commit `ffc9c6d` as RED: eleven overall code smells, comprising ten `java:S6878` and one `java:S1612`. Run `./mvnw -Dtest=Seed4JCommandsFactoryTest,ModuleSetPlanningApplicationServiceTest,Seed4JModuleSetCatalogTest test` before and after the edits. Use `Seed4JCommandsFactoryTest` as the public CLI checkpoint. Run Habit Hooks on the four changed files, then `./mvnw test`, `npm run prettier:check`, a post-GREEN `refactor-design` review, and a local Sonar analysis followed by API confirmation that overall `code_smells` is zero. Commit and push the focused fix so PR #319 reruns.

#### Acceptance Criteria

All existing focused and public tests remain green, the generated apply-set diagnostics remain unchanged, local Sonar reports zero overall code smells for the new branch head, and PR #319 contains the focused follow-up commit.

## Progress

- [x] Repository instructions and required skills loaded.
- [x] Branch, clean worktree, recent commit convention, relevant files, and Habit Hooks availability audited.
- [x] New living ExecPlan created.
- [x] Milestone 1 RED confirmed.
- [x] Milestone 1 GREEN, Habit Hooks analysis, focused validation, and commit `df92fc8` completed.
- [x] Milestone 2 GREEN, Habit Hooks analysis, focused validation, and commit `1cf44fa` completed.
- [x] Milestone 3 GREEN, Habit Hooks analysis, focused validation, and commit `a716a01` completed.
- [x] Milestone 4 GREEN, Habit Hooks analysis, focused validation, and commit `81bda0f` completed.
- [x] Milestone 5 GREEN, Habit Hooks analysis, focused validation, and commit `f301979` completed.
- [x] Milestone 6 GREEN, Habit Hooks analysis, focused validation, and commit `88e5bfa` completed.
- [x] Milestone 7 GREEN, Habit Hooks analysis, focused validation, and commit `ffc9c6d` completed.
- [x] Final `habit-hooks --branch` findings analyzed and enforced findings resolved.
- [x] Final `./mvnw test` passed with 536 tests.
- [x] Final `npm run prettier:check` passed.
- [x] Post-GREEN `refactor-design` review completed and recorded.
- [x] User asked to run `./mvnw clean verify` and return exit code plus concise relevant failure summary.
- [x] Milestone 8 Sonar RED confirmed at 11 overall code smells on `ffc9c6d`.
- [x] Milestone 8 focused behavior suite and public checkpoint are GREEN with 91 tests.
- [x] Milestone 8 Habit Hooks findings analyzed; no changed-file findings.
- [x] Milestone 8 post-GREEN design review and documentation reconciliation completed.
- [x] Milestone 8 final tests, formatting, and local Sonar validation passed: 536 tests, Prettier clean, zero overall code smells.
- [x] Milestone 8 committed as `faed40e` and pushed to PR #319.
- [x] PR #319 CI passed on `faed40e`: 100% coverage and zero bugs, vulnerabilities, security hotspots, duplications, and code smells.

## Decisions

- Decision: Deliver exactly seven independently focused commits in the user-specified order.
  Rationale: The order first removes the confirmed observable defect, then performs behavior-preserving responsibility extractions whose checkpoints isolate regressions.
  Date/Author: 2026-08-24 / Codex
- Decision: Observe all new or adjusted behavior through existing public CLI, bootstrap-primary, launcher, or persistence test homes.
  Rationale: Extracted package-private components are implementation topology and do not justify structural tests.
  Date/Author: 2026-08-24 / Codex
- Decision: Keep runtime DTOs, architectural markers, and apply/apply-set domain flows separate even where technical metadata is shared.
  Rationale: Similar representations do not erase bounded-context or architectural ownership.
  Date/Author: 2026-08-24 / Codex
- Decision: Make `BashCompletionCommand` a reusable specification factory and make a private `BashCompletionInvocation` the Picocli `Callable`.
  Rationale: The Spring component retains only the stable installation service, while each built tree owns its `CommandSpec`, option models, parsed values, and execution lifecycle.
  Date/Author: 2026-08-24 / Codex
- Decision: Preserve the five-argument public `JavaProcessChildLauncher` constructor and create the package-private request factory inside that construction boundary.
  Rationale: Passing a package-private secondary type from the separate pre-Spring composition package is impossible in Java, while changing or supplementing the existing public constructor would violate the explicit public-API and no-convenience-overload constraints. The composition therefore remains source-compatible and continues to own all concrete collaborator selection.
  Date/Author: 2026-08-24 / Codex
- Decision: Treat the local Sonar overall metric as the RED gate and reuse existing behavior suites instead of adding tests for record-pattern or method-reference syntax.
  Rationale: The requested changes are behavior-preserving language-level simplifications; a new test could only mirror implementation topology, which the repository explicitly forbids.
  Date/Author: 2026-08-24 / Codex

## Risks and Mitigations

- Risk: Picocli stores parsed values inside mutable specifications and user objects, so reusing either across roots can preserve stale state.
  Mitigation: Create both the invocation owner and its `CommandSpec` per `spec()` call and prove independence by executing the older root after constructing the newer one.
- Risk: Refactors can subtly reorder Bash candidates, JVM arguments, or serialized output.
  Mitigation: Preserve sorted data structures and existing rendering code while moving one responsibility at a time; run the complete focused behavior suites after every milestone.
- Risk: Java records containing maps or lists are only shallowly immutable.
  Mitigation: Take defensive immutable copies at the candidate/request ownership boundaries and preserve deterministic ordering with sorted immutable snapshots.
- Risk: Atomic-move fallback and cleanup can mask the original exception or change replacement semantics.
  Mitigation: Match both existing call paths, treat cleanup as best effort after a failed publication, and retain the original thrown failure as the caller-visible exception.
- Risk: The large integration test class can make focused cycles slower and expose unrelated environment failures.
  Mitigation: Run the full named suite required by the TDD profile, stop on unrelated failures, and retain output needed for diagnosis without running the prohibited full verify gate.
- Risk: The living ExecPlan directory may be ignored by Git and therefore absent from the seven source commits.
  Mitigation: Maintain the requested file in the shared workspace throughout execution and report its final path explicitly.
- Risk: Local Sonar can show zero new issues while still containing overall issues when its first analysis establishes the current commit as the new-code baseline.
  Mitigation: Validate through `/api/measures/component` using the overall `code_smells` metric, matching `tests-ci/sonar.sh`, and require an overall value of zero.

## Validation Strategy

Use the milestone-specific focused suites as the RED/GREEN and public-path checkpoints. Run Habit Hooks on each changed file set and record every finding, resolving enforced findings without snoozing. After the original seven commits, run `habit-hooks --branch`, `./mvnw test`, and `npm run prettier:check`. For the Sonar follow-up, keep the existing behavior tests green, run the same final test/format gates, and submit a local Sonar analysis whose processed overall `code_smells` metric is zero. Enter each `refactor-design` review only after behavior and public-path gates are green; rerun affected tests for any behavior-preserving consolidation. Do not run `./mvnw clean verify`; ask the user to run it locally and provide the exit code plus a concise summary of relevant failures.

## Documentation Impact

`README.md` and the files under `documentation/` describe public commands and architecture. They require no content change because option spelling, help text, messages, output, runtime boundaries, persistence behavior, and module-set planning semantics are unchanged, and the new package-private types follow the already documented primary/secondary and pre-Spring composition rules. This ExecPlan is the only maintenance documentation updated for the Sonar follow-up and records implementation decisions, tests, Habit Hooks findings, review outcomes, and the overall Sonar metric.

## Rollout and Recovery

The original seven commits are intentionally independently revertible in reverse order, and the Sonar cleanup will be a separate eighth commit. If focused validation reveals a regression, stop before committing, restore GREEN within the current milestone, and record the finding here. After commits exist, recovery can revert only the implicated commit without rewriting branch history; no data migration, configuration rollout, or user action is required.

## Lessons Learned

The branch began clean at `bda3792`, and recent commits consistently use English Conventional Commit subjects with explanatory bodies for substantive fixes/refactors. Habit Hooks is available at `/home/renanfranca/.local/bin/habit-hooks`; snoozing remains explicitly unauthorized.

The Milestone 1 focused RED ran 83 tests and failed only `shouldKeepEarlierBashCompletionOptionsAfterBuildingAnotherCommandTree`. Its output contained the separated value-completion branch even though the earlier tree parsed `--no-complete-values`, proving that `BashCompletionCommand.call()` read the later tree's default-valued `commandSpec` field.

Milestone 1 GREEN ran the same 83 tests successfully. Habit Hooks initially reported the new invocation `call()` as oversized because it combined generation, installation, error translation, and instructions; extracting the cohesive installation path and instruction rendering removed the changed-file finding while the suite remained green. The scan also repeated repository-wide duplication findings for architectural markers, cross-context runtime DTOs, pre-Spring test composition, and generic test fixtures/assertions. Those are analyzed as intentional bounded-context separation or explicitly excluded cleanup and were neither changed nor snoozed.

Milestone 2 began and ended with the same 30 focused launcher/bootstrap-primary tests green. The extracted request defensively copies properties and arguments, and request resolution preserves the prior start-class-before-overlay failure order. Habit Hooks reported no issue in either new package-private type and repeated pre-existing test-fixture duplication. It also reported the unchanged five-parameter public launcher constructor; this is classified as no action because eliminating it would require the forbidden public API change or an inaccessible package-private parameter, while the mixed runtime-resolution responsibility that motivated the milestone has been removed.

Milestone 3 RED ran 75 `Seed4JCommandsFactoryTest` tests and failed only `shouldShowHelpWithoutReadingRuntimeDisplay`. The failure originated while `buildCommandSpec()` eagerly called the unavailable runtime reader, before Picocli could execute `--help`, which confirms the intended lazy-version behavior is absent.

Milestone 3 GREEN ran all 75 factory tests successfully. Picocli now receives a Spring-managed `IVersionProvider`, so help and other commands build without runtime access while the existing standard, extension, build-metadata, and fallback version scenarios continue to render the moved text unchanged. Habit Hooks reported no finding in either changed production class; its fixture/test findings were pre-existing size, parameter-count, and generic duplication concerns explicitly outside this work.

Milestone 4 GREEN ran 84 Bash generator/factory tests successfully. `BashCompletionCandidateCollector` now owns recursive Picocli traversal, and `BashCompletionCandidates` takes defensive immutable list copies plus unmodifiable `TreeMap` snapshots so the renderer retains the prior deterministic iteration order. Habit Hooks reported no finding in any of the three changed production files; no topology tests were added.

Milestone 5 GREEN ran the same 84 Bash generator/factory tests successfully, including execution of the generated Bash for attached, separated, quoted, filtered, empty, and disabled value completion. The template, case rendering, quoting, and completion algorithm moved unchanged into `BashCompletionScriptRenderer`, leaving the generator as collector/renderer coordination. Habit Hooks reported no finding in either changed production file; no renderer-topology test was added.

Milestone 6 began and ended with the same six writer/repository tests green. `AtomicFilePublisher` centralizes adjacent UUID temporary paths, content/source staging, atomic replacement with fallback, failure cleanup, and `IOException` propagation while both callers keep their public construction and error translation. Habit Hooks reported no finding in any of the three changed production files; the existing caller-facing tests continue to cover cleanup and fallback without an internal publisher test.

Milestone 7 began and ended with the same 85 public command, apply/apply-set, and Bash-completion tests green. `ProjectPathOptionSpecFactory` centralizes the option spelling and existing Picocli presentation/parsing metadata while building a fresh mutable `OptionSpec` on every call; `ApplyModuleSubCommand` still maps to `ProjectPath`, and `ApplyModuleSetCommand` still maps independently to `ModuleSetProjectPath`. Habit Hooks reported no finding in the new factory. It repeated pre-existing import-block duplication plus size and responsibility findings in `ApplyModuleSubCommand`; those are analyzed as unrelated to the focused metadata extraction and changing them would expand the explicitly bounded seven-commit scope. Generic test duplication remains explicitly excluded, and no finding was snoozed.

The final `habit-hooks --branch` completed with exit code 1 because it reports advisory findings across the repository: 271 duplicated-code pairs, eight oversized files, 51 oversized functions, seven parameter-count findings, and one swallowed exception in an unchanged test. Branch-touched references were classified individually: the public five-argument launcher constructor is intentionally retained for API compatibility; `ApplyModuleSubCommand` size/function findings predate and are independent of the project-path metadata seam; `CliFixture`, `Seed4JCommandsFactoryTest`, bootstrap fixtures, and other test findings fall under the explicitly excluded generic fixture/assertion cleanup; import-block matches are not a shared business concept; and bounded-context DTO/marker duplication remains intentional. No new factory, request, collector, renderer, provider, or publisher received an actionable finding, no enforced changed-code finding remains, and no snooze was used.

Final validation ran `./mvnw test`: 536 tests passed with no failures, errors, or skips. `npm run prettier:check` also passed. The post-GREEN `refactor-design` review found no further behavior-preserving change worth applying: mutable Picocli state is invocation-owned, extracted request/candidate collections own immutable snapshots, responsibilities are separated at named primary/secondary seams, and no infrastructure representation leaked into application/domain or crossed the bootstrap/command boundary. The remaining constructor arity, two-file extension publication boundary, and large legacy apply adapter are established adjacent constraints rather than regressions introduced by these commits.

The PR #319 Sonar diagnosis showed that the CI and local server both analyzed `ffc9c6d` with SonarQube Community Build `26.8.0.126808` and found eleven overall smells. The apparent local zero was the `new_code_smells` metric after the first analysis established the same commit as baseline; the authoritative overall `code_smells` metric and the PR log both reported eleven. Ten findings are `java:S6878` record-pattern opportunities across four module-set files, and one is the `java:S1612` method-reference opportunity in the planning-problem renderer.

Milestone 8 used the existing local Sonar result as its non-behavioral RED and kept the public behavior suite as the contract guard. The baseline and GREEN runs each exercised 91 tests across `Seed4JCommandsFactoryTest`, `ModuleSetPlanningApplicationServiceTest`, and `Seed4JModuleSetCatalogTest`; the GREEN run passed after adding the missing import required by the method reference. Habit Hooks reported no findings in any of the four changed production files.

The Milestone 8 `refactor-design` review classified the changed representation as no further action: each record pattern binds only the components already read by its branch, introduces no state or temporal coupling, preserves exhaustive sealed-hierarchy mapping, and leaves domain, primary rendering, and secondary history responsibilities in their original layers. `README.md` and `documentation/` need no update because no option, diagnostic, workflow, persisted representation, or architectural boundary changed.

Final Milestone 8 validation ran `./mvnw test` with 536 passing tests and `npm run prettier:check` successfully. The local Sonar submission completed on Community Build `26.8.0.126808`; after asynchronous processing, `/api/measures/component` reported `code_smells=0`, `violations=0`, and the open code-smell issue search returned zero. The local coverage display was 99.3% because the aggregate XML retained line numbers from the earlier user-run `clean verify`; Sonar explicitly reported that stale line mismatch, while the fresh unit-test report and every behavior test were green. The PR CI recreates its workspace and previously produced 100% coverage, so no coverage code change is indicated by this local diagnostic artifact.

Commit `faed40e` (`refactor(command): destructure module set records`) contains only the four Sonar-targeted production files. A second processed Sonar analysis attached the zero-smell result to the committed revision `faed40ed798540e9bc3a01b5493c597e4bd38cbb`, confirming `code_smells=0`, `violations=0`, and zero open code-smell issues before publication.

PR workflow run `32766347418` completed successfully in 3m34s. Its fresh local Sonar analysis reported 100.0% coverage, zero bugs, zero vulnerabilities, zero security hotspots, 0.0% duplicated-line density, and zero overall code smells, confirming that the cleanup behaves correctly outside the developer workstation and that the earlier local coverage discrepancy was stale-report state only.
