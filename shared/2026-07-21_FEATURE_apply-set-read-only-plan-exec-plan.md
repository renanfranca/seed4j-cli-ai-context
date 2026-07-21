# Implement read-only planning for module sets

This ExecPlan is a living document. Keep `Progress`, `Decisions`, `Risks`, and `Lessons Learned` up to date as work advances.

## Purpose / Big Picture

Issue #296 adds `seed4j apply-set <slug>... --plan`, a read-only command that explains how an explicitly requested set of visible Seed4J modules would be ordered, which dependencies are satisfied or missing, and which parameters would be used. A valid plan exits with code 0; malformed requests and plans with predictable validation problems exit with code 2. The command must never generate files, create `.seed4j` or `.git`, append history, or create commits.

This work is limited to authorized feature maintenance in this repository. The parent issue #295 supplies architectural direction only; applying a planned set, commits, rollback, presets, JSON output, and changes to `seed4j apply <module>` remain outside this plan.

## Scope

In scope: register `apply-set`; require one or more positional visible module slugs and `--plan`; accept `--project-path` with default `.`; expose the union of visible module property options without `--commit`; trust the Seed4J business invariant that one property key has one type across modules and preserve that type through Picocli; reject unknown, hidden, or duplicate requested slugs; calculate execution order exclusively with `Seed4JLandscape.sort(...)`; recursively validate module and feature dependencies against project history and explicitly requested earlier modules; reconcile selected property definitions; resolve values with explicit CLI input over project history over default; reject known options irrelevant to the selected set; aggregate predictable problems; render the complete plan; document the command and architecture; and prove read-only behavior.

Out of scope: module application, `--commit`, Git initialization, commits, rollback, partial failure recovery, presets, JSON or other structured formats, provider auto-selection, and any behavior change to the existing single-module `apply` command.

## Definitions

`Requested modules` are the visible module slugs supplied by the caller, retained in caller order even when already present in history. `Execution order` is the requested set reordered only by `Seed4JLandscape.sort(...)`. A `module dependency` names one exact prerequisite module. A `feature dependency` names a capability that may have multiple provider modules; a provider must be explicitly requested or already present in history. `Visible module` means a resource returned by the existing visible catalog exposed by `Seed4JModulesApplicationService.resources()`; hidden resources are neither requestable nor suggested. `History` means prior project actions and their latest parameters read through `ProjectsApplicationService`. `Property type invariant` means that the same property key represents the same Seed4J type in every module; like the Seed4J Vue application, the CLI trusts this business convention rather than validating it. `Reconciled property` is one property key shared by one or more selected modules after mandatory status, default, and description rules have been combined. A `valid reusable plan` is an immutable domain result whose execution order and validations need not be recalculated by a future mutable issue.

## Existing Context

The current root command is assembled by `src/main/java/com/seed4j/cli/command/infrastructure/primary/Seed4JCommandsFactory.java` from Spring-discovered `Seed4JCommand` components. `ApplyModuleCommand` dynamically registers one `ApplyModuleSubCommand` per visible module. `ApplyModuleSubCommand` currently owns Picocli option construction, raw option conversion, history reads, dependency planning, single-module plan rendering, and application calls. `ApplyModuleDependencyPlanner` and `ApplyModuleParameterResolver` are primary-adapter implementation types built around Seed4J core resources.

Seed4J core 2.2.0 exposes visible `Seed4JModulesResources`, a `Seed4JLandscape` whose public `sort(Collection<Seed4JModuleSlug>)` defines deterministic order, module resources with organization/dependencies/property definitions, and `ProjectHistory` with applied actions and latest parameters. The new flow must wrap the concrete core application services behind CLI-owned domain capability interfaces so application code does not depend on concrete adapters, filesystem behavior, Spring, or `..infrastructure..` packages.

Issue #296 requires a read-only multi-module plan and issue #295 requires the validated plan to be reusable by later application work. Repository rules require Types Driven Development, explicit hexagonal boundaries, behavior-focused tests, Java 25, Prettier formatting, and `./mvnw test` as the agent-side final gate. `./mvnw clean verify` must be requested from the user rather than run automatically.

## Desired End State

`seed4j apply-set init maven-java --project-path . --plan` prints `Requested modules`, `Execution order`, `Dependency validation`, `Resolved parameters`, any missing-parameter or validation-problem sections, a final valid/invalid status, and `No changes were applied.`. Reversing the positional input preserves the requested display but not the landscape-derived execution order. Every predictable problem is collected after the requested set can be resolved. Invalid usage and invalid plans write clear diagnostics to stderr and return 2; a complete plan returns 0.

Production code contains an immutable CLI-owned domain model for request, requested set, plan, dependency status, resolved parameters, and validation problems; a `ModuleSetPlanningApplicationService`; domain ports for the visible module catalog/landscape and history; secondary adapters around the two Seed4J core application services; a primary `ApplyModuleSetCommand` plus renderer; and a shared Picocli module-option factory used by both `apply` and `apply-set` without changing existing `apply` behavior.

## Milestones

### Milestone 1 - Establish the public CLI contract

#### Goal

Register `apply-set`, expose help/completion and module property options, require one or more slugs plus `--plan`, omit `--commit`, and aggregate duplicate/unknown/hidden slug diagnostics.

#### Changes

- Edit `src/test/java/com/seed4j/cli/command/infrastructure/primary/Seed4JCommandsFactoryTest.java` and its fixture, or add one behavior-oriented `ApplyModuleSetCommandTest.java` only if the root suite becomes unclear, to specify public CLI behavior.
- Add `src/main/java/com/seed4j/cli/command/infrastructure/primary/ApplyModuleSetCommand.java` and wire it through Spring/root command discovery.
- Extract shared module property option construction from `ApplyModuleSubCommand` into a primary-adapter factory so option names, types, descriptions, parameter labels, defaults used for completion, and candidates remain compatible.

#### Validation

- Command: `./mvnw -Dtest=Seed4JCommandsFactoryTest test`
- Expected result: the relevant suite first fails because `apply-set` is absent, then passes with help, completion, required arguments, and error exit codes covered.

#### Acceptance Criteria

- `seed4j --help` and completion include `apply-set`.
- `apply-set --help` exposes `--project-path`, mandatory `--plan`, global module properties, and no `--commit`.
- Missing slugs, missing `--plan`, duplicate slugs, unknown slugs, and hidden slugs return 2 with clear stderr diagnostics.

### Milestone 2 - Build the reusable domain plan and deterministic dependency validation

#### Goal

Move multi-module planning into domain/application boundaries, order only through the Seed4J landscape, preserve requested order separately, and validate recursive dependencies against earlier requested modules and history.

#### Changes

- Add immutable value objects and records under `src/main/java/com/seed4j/cli/command/domain/moduleset/` for slugs, project path/request, requested set, module catalog metadata, plan, validity, dependency lines/statuses, parameter definitions/results, and validation problems.
- Add business-capability ports under the same domain package for reading the visible module catalog/landscape order and project planning history.
- Add `src/main/java/com/seed4j/cli/command/application/ModuleSetPlanningApplicationService.java` to orchestrate resolution, ordering, dependency validation, and property resolution without concrete adapter dependencies.
- Add secondary adapters under `src/main/java/com/seed4j/cli/command/infrastructure/secondary/` wrapping `Seed4JModulesApplicationService` and `ProjectsApplicationService`, including conversion between Seed4J core types and CLI-owned domain types.
- Add or extend behavior tests under `src/test/java/com/seed4j/cli/command/application/` for the stable application-service contract, using simple in-memory port implementations rather than mocks where practical.

#### Validation

- Command: `./mvnw -Dtest=ModuleSetPlanningApplicationServiceTest test`
- Command: `./mvnw -Dtest=Seed4JCommandsFactoryTest,ModuleSetPlanningApplicationServiceTest test`
- Expected result: inverted input has stable landscape execution order; recursive dependency lines are deduplicated and identify requiring requested modules; module and feature dependencies are satisfied only by history or explicit earlier requested modules; missing feature providers list visible candidates alphabetically and invalidate the plan.

#### Acceptance Criteria

- Requested modules already in history remain in the execution plan.
- The application service never selects a feature provider implicitly.
- The returned immutable plan contains all order and validation data needed by a future application issue.

### Milestone 3 - Reconcile and resolve properties with aggregated validation

#### Goal

Represent shared selected properties once, preserve deterministic display order, resolve sources correctly, and report all semantic conflicts, irrelevant explicit options, and missing mandatory values together.

#### Changes

- Extend the module-set domain model and application service with property reconciliation by key: trust the shared type declared by the first definition; mandatory if any definition is mandatory; at most one distinct default and description; first occurrence order from execution order and module declaration order.
- Preserve explicit CLI value types produced by Picocli. Do not add a neutral text type, late conversion, or type-conflict validation; the same-key/same-type rule is a Seed4J business invariant shared with the Vue application.
- Track which known property options were explicitly supplied and invalidate options unused by selected modules.
- Extend the primary renderer to show resolved source, missing parameters, dependency problems, semantic conflicts, irrelevant options, final status, and the read-only footer.
- Add focused behavior cycles to the existing application and CLI integration suites.

#### Validation

- Command: `./mvnw -Dtest=ModuleSetPlanningApplicationServiceTest,Seed4JCommandsFactoryTest test`
- Expected result: unions and shared properties render once; CLI overrides history which overrides default; optional defaults remain informational; all missing mandatory values and all other predictable problems appear; conflicting defaults or descriptions and irrelevant explicit options return 2.

#### Acceptance Criteria

- Global property definitions retain Picocli's current type and completion metadata from their first occurrence.
- The planner does not validate the business invariant that repeated property keys have the same type.
- The plan aggregates rather than short-circuits predictable post-resolution problems.

### Milestone 4 - Prove read-only behavior and compatibility

#### Goal

Exercise the command through the CLI against empty existing and nonexistent project paths, prove the filesystem and Git state are unchanged, and keep the existing `apply` suite green.

#### Changes

- Add CLI integration scenarios that snapshot directory/nonexistence state before and after valid and invalid plans.
- Verify no generated files, `.seed4j`, `.git`, history actions, or commits appear.
- Refactor only while behavior suites stay green and remove any implementation-topology tests made unnecessary by the public scenarios.

#### Validation

- Command: `./mvnw -Dtest=Seed4JCommandsFactoryTest test`
- Expected result: valid and invalid `apply-set` plans are observably read-only, including for a nonexistent path, and all existing single-module `apply` tests still pass.

#### Acceptance Criteria

- Planning does not require creating the target directory or history file.
- Existing `seed4j apply <module>` output and exit-code behavior remain compatible.

### Milestone 5 - Document, format, and run the agent validation gate

#### Goal

Document user-visible semantics and the new hexagonal flow, format all supported files, and validate the full test suite.

#### Changes

- Update `README.md` with syntax, exit codes, read-only guarantees, and differences from single-module `--plan`.
- Update `documentation/Commands.md` with full examples, validation behavior, property/dependency rules, and scope limits.
- Update `documentation/hexagonal-architecture.md` with primary command to application service to domain ports to secondary adapter flow.
- Keep this ExecPlan's progress, decisions, risks, validation results, and lessons current.

#### Validation

- Command: `npm run prettier:format`
- Command: `npm run prettier:check`
- Command: `./mvnw test`
- Expected result: formatting checks and all JUnit tests pass. The user then runs `./mvnw clean verify` locally and reports its exit code plus any relevant failure summary.

#### Acceptance Criteria

- Documentation clearly distinguishes invalid multi-module plan exit code 2 from single-module plan's informational exit code 0.
- Agent-side formatting and test gates pass without running `clean verify` automatically.

### Milestone 6 - Close coverage with observable contracts and remove unsupported type reconciliation

#### Goal

Close the apply-set coverage gaps through CLI and planning behaviors, remove dead renderer paths, and align property typing with the Seed4J Vue precedent without changing the core repository.

#### Changes

- Add behavior cases for duplicate-only requests, missing catalog dependencies, and requested feature providers ordered after their consumers.
- Add CLI behavior for boolean Picocli values and conflicting default/description diagnostics using simple in-memory catalog collaborators.
- Remove `TEXT`, late explicit-value conversion, invalid-value planning problems, and selected type-conflict validation.
- Replace the renderer's unreachable problem-label switch arms with the exact set of problems rendered in the general validation section.
- Update README, command documentation, and this ExecPlan to state the property type invariant explicitly.

#### Validation

- Command: `./mvnw -Dtest=ModuleSetPlanningApplicationServiceTest,Seed4JCommandsFactoryTest test`
- Command: `npm run prettier:format`
- Command: `npm run prettier:check`
- Command: `./mvnw test`
- Expected result: behavior suites pass and the unit JaCoCo report shows zero missed lines and branches for `ModuleSetPlanner`, `ApplyModuleSetPlanRenderer`, and `ModulePropertyOptionSpecFactory`.

#### Acceptance Criteria

- Picocli remains the only CLI string-to-typed-value converter for globally registered module properties.
- Default and description conflicts remain invalid; repeated-key type consistency is trusted rather than checked.
- No coverage-only test directly exercises an internal mapper, converter, or renderer helper.

### Milestone 7 - Replace command composition with Spring-managed secondary adapters

#### Goal

Remove the artificial `command.composition` wiring while preserving the domain ports and every observable `apply-set` behavior. Spring is already available in the command runtime, so the secondary adapters should be ordinary Spring components that integrate the external Seed4J application APIs directly.

#### Changes

- Keep `ModuleSetCatalog` and `ModuleSetPlanningHistoryReader` as domain capability interfaces.
- Make `Seed4JModuleSetCatalog` and `ProjectsModuleSetPlanningHistoryReader` Spring components with direct constructor injection of `Seed4JModulesApplicationService` and `ProjectsApplicationService`.
- Remove `ModuleSetPlanningConfiguration` and the adapter-internal functional interfaces `Seed4JModulesResourcesReader`, `Seed4JLandscapeModuleSorter`, and `ProjectsHistoryReader`.
- Update the CLI fixture to construct the same production adapter API used by Spring.
- Refine `HexagonalArchTest` so secondary adapters cannot depend on their own context's application layer while integration with an external application API remains allowed.
- Clarify in `AGENTS.md` and `documentation/hexagonal-architecture.md` that explicit composition packages are reserved for pre-Spring bootstrap work.

#### Validation

- Command: `./mvnw -Dtest=HexagonalArchTest,ModuleSetPlanningApplicationServiceTest,Seed4JCommandsFactoryTest test`
- Command: `npm run prettier:format`
- Command: `npm run prettier:check`
- Command: `git diff --check`
- Command: `./mvnw test`
- Command: `./mvnw package -q`, followed by a valid `apply-set` invocation through the packaged JAR.
- Expected result: architecture and behavior suites pass, the Spring context discovers both secondary adapters, and the packaged command returns 0 with `Status: VALID` without creating project state.

#### Acceptance Criteria

- No `command.composition` package or adapter-internal method-reference interfaces remain.
- Application and domain still depend only on `ModuleSetCatalog` and `ModuleSetPlanningHistoryReader`.
- CLI output, exit codes, dependency ordering, parameter resolution, and read-only guarantees remain unchanged.

## Progress

- [x] Issue #296 and parent issue #295 inspected on 2026-07-21.
- [x] Repository command flow, existing single-module planner, project history API, and Seed4J landscape API inspected.
- [x] ExecPlan created before production implementation.
- [x] Milestone 1 started.
- [x] Milestone 1 completed.
- [x] Milestone 2 started.
- [x] Milestone 2 completed.
- [x] Milestone 3 started.
- [x] Milestone 3 completed.
- [x] Milestone 4 started.
- [x] Milestone 4 completed.
- [x] Milestone 5 started.
- [x] Milestone 5 completed.
- [x] Milestone 6 started.
- [x] Milestone 6 completed.
- [x] Milestone 7 started.
- [x] Milestone 7 completed.

## Decisions

- Decision: Treat duplicate requested slugs as usage errors rather than silently normalizing them.
  Rationale: The requested issue refinement explicitly chooses rejection and requires an exit code of 2.
  Date/Author: 2026-07-21 / Codex
- Decision: Keep the existing single-module planner and its exit semantics unchanged; build a separate multi-module application/domain flow.
  Rationale: Issue #296 is additive and explicitly excludes changing `seed4j apply <module>`.
  Date/Author: 2026-07-21 / Codex
- Decision: Use CLI-owned domain snapshots at the ports rather than passing Seed4J core resources or filesystem layouts into application/domain.
  Rationale: Repository architecture rules prohibit application dependencies on concrete adapters and require business concepts before values cross inward; adapters can translate the external core API.
  Date/Author: 2026-07-21 / Codex
- Decision: Derive execution order by calling `Seed4JLandscape.sort(...)` in the secondary catalog adapter and never reproduce its algorithm in CLI code.
  Rationale: The issue makes the core landscape the exclusive ordering authority.
  Date/Author: 2026-07-21 / Codex
- Decision: Expose global property metadata through `ModuleSetPlanningApplicationService.availableProperties()` and keep the new primary command free of concrete Seed4J core application services.
  Rationale: Picocli must construct options before parsing, while the architectural boundary still requires catalog translation through a secondary adapter. The application service is the stable inward-facing source for that read-only metadata.
  Date/Author: 2026-07-21 / Codex
- Decision: Read project history only after duplicate and unknown slug validation succeeds.
  Rationale: Malformed requests can report all pre-planning slug problems without touching the target path or invoking external storage behavior.
  Date/Author: 2026-07-21 / Codex
- Decision: Keep dependency validation lines in the reusable plan with provider, candidates, and all requesting selected modules, and derive validity from typed problems rather than renderer text.
  Rationale: A future mutable issue can execute the already ordered set and inspect the same validated facts without parsing output or recalculating decisions.
  Date/Author: 2026-07-21 / Codex
- Decision: Put dependency and property rules in the pure `ModuleSetPlanner` domain service, with `ModuleSetPlanningApplicationService` as a thin orchestration facade.
  Rationale: Dependency satisfaction, property reconciliation, value precedence, and plan validity are business rules. Keeping them in domain preserves the repository's application/domain boundary while retaining the requested application-service API.
  Date/Author: 2026-07-21 / Codex
- Decision: Wire external Seed4J and project application services through `command.composition` into technical secondary readers.
  Rationale: Secondary adapters must implement domain ports without depending directly on another context's application layer, and primary adapters must not construct secondary adapters. The explicit composition root supplies method references while the secondary adapters retain all translation logic.
  Date/Author: 2026-07-21 / Codex
- Decision: Supersede the preceding `command.composition` decision and use Spring-managed secondary adapters that inject the external Seed4J application APIs directly.
  Rationale: Spring is available in the command runtime, the three functional interfaces only relocate the same coupling, and composition packages in this repository are reserved for the pre-Spring bootstrap boundary. The architecture rule should forbid a secondary adapter from depending on its own application layer without blocking an adapter from integrating an external application API.
  Date/Author: 2026-07-21 / Codex
- Decision: Treat same-key/same-type consistency as a Seed4J business invariant rather than a runtime validation owned by the CLI.
  Rationale: The Seed4J Vue application deduplicates selected definitions by key and trusts their type while its primary adapter casts user input. The CLI will follow the same model: Picocli casts input, while the planner continues to reconcile mandatory status, defaults, and descriptions only.
  Date/Author: 2026-07-21 / Codex

## Risks and Mitigations

- Risk: A custom extension could violate the same-key/same-type business invariant.
  Mitigation: Match the existing Seed4J Vue behavior and document that such a catalog is outside the supported contract; use the first visible definition to register the single global Picocli option.
- Risk: Reading history for a nonexistent target could cause the external repository to create storage or fail before a plan can be rendered.
  Mitigation: Characterize `ProjectsApplicationService.getHistory` through a focused public-path test; if it mutates or rejects missing paths, keep existence/layout handling inside the secondary history adapter and return empty domain history without writing.
- Risk: Reusing core types in the new application service would invert the required hexagonal boundary.
  Mitigation: Keep core imports inside Spring-managed secondary translation adapters; application and domain compile only against CLI-owned immutable types and capability interfaces.
- Risk: Broad integration tests could become slow and obscure small TDD failures.
  Mitigation: Specify domain/application behaviors through the stable application-service API and run the CLI public path at least every two cycles, with the complete existing root suite as the compatibility checkpoint.
- Risk: Existing untracked `_temporary/` content belongs to the user.
  Mitigation: Add and update only the explicitly requested ExecPlan file; do not delete, stage, or rewrite unrelated temporary content.
- Risk: Broadening the architecture rule could accidentally allow a secondary adapter to call its own application service.
  Mitigation: Scope the rule per business context and retain the existing cross-context rules; only external application APIs remain valid secondary dependencies.

## Validation Strategy

Use strict behavior TDD in small RED-GREEN-REFACTOR cycles. Each cycle adds one observable behavior at the highest useful boundary, predicts and confirms the expected failure, implements the minimum passing behavior, and refactors only while green. Run the full relevant behavior suite each cycle and a root CLI checkpoint at least every two cycles. After functional completion run `npm run prettier:format`, `npm run prettier:check`, and `./mvnw test`. Do not run `./mvnw clean verify`; request that gate from the user.

Manual checks will execute representative valid and invalid `apply-set` commands against temporary existing-empty and nonexistent paths and compare pre/post directory state. Tests must explicitly assert stdout/stderr separation, exit code 0 versus 2, and absence of `.seed4j`, `.git`, generated files, history updates, and commits.

## Rollout and Recovery

The feature is additive and read-only, so rollout consists of shipping the new command while retaining `apply`. If a regression is found, revert the new command registration, module-set domain/application types, adapters, renderer, tests, and documentation as one focused change; the pre-existing single-module path is deliberately not migrated and remains the recovery path. No data migration or persisted-format rollback is required because this issue writes no state.

## Lessons Learned

- Seed4J core 2.2.0 already exposes the required ordering authority as `Seed4JLandscape.sort(Collection<Seed4JModuleSlug>)`; its result can be translated into a CLI-owned ordered requested set.
- The current single-module command combines primary parsing, external service reads, dependency discovery, rendering, and mutation. The new flow is an opportunity to introduce the requested boundary without destabilizing the existing command.
- The public issue body is less specific than the refined implementation prompt about duplicate rejection, invalid-plan exit code 2, irrelevant known options, requiring-module attribution, and nonexistent paths; this ExecPlan treats the refined prompt as the controlling acceptance contract. The initially proposed global type-conflict handling was later superseded by the documented same-key/same-type Seed4J invariant.
- Seed4J's Vue application deduplicates selected properties by key and uses the retained definition type to cast form values; it does not validate type conflicts across modules. The CLI now follows that same business convention while retaining stricter default and description reconciliation required by issue #296.
- `ProjectsApplicationService.getHistory` returns an empty history for both an existing empty directory and a nonexistent project path without creating directories, `.seed4j`, or `.git`; integration tests now lock down that read-only behavior.
- Picocli's programmatic variadic positional did not reject an absent list before invoking `Callable`; the command therefore performs an explicit empty-list usage check and returns code 2 instead of leaking a null dereference.
- The architecture test prohibits secondary adapters from depending on any application package, including Seed4J core contexts, and prohibits primary adapters from constructing secondary adapters. An explicit `command.composition` root resolves both directions while keeping the new command dependent only on its application service.
- Full-repository Prettier initially identified seven pre-existing Java files that differed from the installed formatter. The requested `npm run prettier:format` normalized those files mechanically, after which `npm run prettier:check` passed.
- The remaining apply-set coverage gaps represented observable edge cases (duplicate-only requests, missing catalog modules, late feature providers, typed Picocli input, and rendered property conflicts) plus dead type-conversion and renderer-label paths. Behavior tests now cover the former, while trusting the Seed4J type invariant removed the latter instead of preserving code solely for coverage.
- During mutation-assisted RED checks, the system clock moved backwards and Maven briefly reused a future-dated mutated class from `target`. A focused `clean test` removed that generated artifact; source files were not reverted or overwritten.
- The broad secondary-to-application architecture rule conflated calls into a secondary adapter's own application layer with integration through an external context's public application API. Scoping the rule by bounded context preserves the inward dependency restriction while allowing a secondary adapter to implement its port through an external service.

## Validation Results

- `./mvnw -q -Dtest=HexagonalArchTest,ModuleSetPlanningApplicationServiceTest,Seed4JCommandsFactoryTest test`: first failed only because `command.composition.ModuleSetPlanningConfiguration` violated the new pre-Spring-only composition rule, then passed after the Spring-managed adapter refactor; 93 tests passed.
- `npm run prettier:format`: passed after the Milestone 7 refactor.
- `npm run prettier:check`: passed with `All matched files use Prettier code style!` after the Milestone 7 refactor.
- `git diff --check`: passed with no whitespace errors after the Milestone 7 refactor.
- `./mvnw test -q`: passed after the Milestone 7 refactor.
- `./mvnw package -q`: passed and produced `target/seed4j-cli-0.0.4-SNAPSHOT.jar`.
- Packaged JAR checkpoint: `java -jar target/seed4j-cli-0.0.4-SNAPSHOT.jar apply-set init --project-path /tmp/seed4j-cli-m7-C2gpos/nonexistent-project --project-name 'Sample application' --base-name sampleApplication --node-package-manager npm --plan` returned 0, rendered `Status: VALID` and `No changes were applied.`, and left the project path nonexistent.
- `./mvnw -Dtest=ModuleSetPlanningApplicationServiceTest,Seed4JCommandsFactoryTest,HexagonalArchTest test -q`: passed after the final domain/composition refactor.
- `npm run prettier:format`: passed; supported files were formatted.
- `npm run prettier:check`: passed with `All matched files use Prettier code style!`.
- `git diff --check`: passed with no whitespace errors.
- `./mvnw test -q` inside the filesystem sandbox: application assertions passed, but 8 unrelated Mockito-based tests errored because Byte Buddy could not self-attach on Java 25.
- `./mvnw -q clean -Dtest=ModuleSetPlanningApplicationServiceTest,Seed4JCommandsFactoryTest test`: passed after all Milestone 6 RED-GREEN-REFACTOR cycles.
- `./mvnw clean test -q` outside that sandbox with the required attach permission after the final coverage refactor: passed, exit code 0; the same suite contains 519 tests.
- Fresh `target/site/jacoco/jacoco.csv` generated by that clean run: `ModuleSetPlanner`, `ApplyModuleSetPlanRenderer`, and `ModulePropertyOptionSpecFactory` each report 0 missed instructions, branches, lines, methods, and classes.
- `./mvnw clean verify`: intentionally not run; the user must run this final local gate and report the exit code plus any relevant failure summary.
