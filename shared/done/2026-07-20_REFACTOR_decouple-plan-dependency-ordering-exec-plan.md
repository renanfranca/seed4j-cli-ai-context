# Decouple plan dependency discovery and landscape ordering

This ExecPlan is a living document. Keep `Progress`, `Decisions`, `Risks`, `Validation`, and `Lessons Learned` up to date as work advances.

## Purpose / Big Picture

`seed4j apply <module> --plan` must list every direct and transitive dependency in the order defined by the Seed4J landscape, regardless of the order in which module operations declare dependencies. After this refactor, modules and abstract features share the same ordering flow: dependency discovery builds a closure, landscape levels order that closure, and only then does the planner render plan lines. Users can observe the result with `apply seed4j-extension --plan`, which must list `feature:java-build-tool`, `module:java-base`, and `module:spring-boot` in that order without changing the project or Git history.

## Scope

In scope: add a CLI-level characterization test for a transitive feature and module closure, refactor `ApplyModuleDependencyPlanner` into separate discovery, ordering, and line-conversion phases, and pass the public `Seed4JLandscape` from `Seed4JModulesApplicationService` through `ApplyModuleSubCommand`.

Out of scope: changing CLI parameters, options, rendering text, dependency status calculations, automatically choosing a concrete module for an abstract feature, publishing a new `seed4j` version, or running `./mvnw clean verify` automatically.

Safety boundary: This task is limited to authorized, defensive maintenance of this repository. Do not provide offensive guidance or policy-bypassing instructions.

## Definitions

`Dependency closure` is the unique set of direct and transitive `Seed4JLandscapeDependency` values required by the requested module.

`Landscape level` is an ordered layer exposed by `Seed4JLandscape.levels()`. Its `slugs()` stream includes ordinary modules, feature slugs, and modules grouped under a feature.

`Feature dependency` is an abstract requirement such as `feature:java-build-tool`; it remains unresolved until a candidate such as `gradle-java` or `maven-java` is selected.

`Plan line` is the rendered dependency token and its existing applied, pending, satisfied, or pending-choice status.

## Existing Context

This plan builds on `shared/done/2026-07-20_FIX_plan-dependency-order-exec-plan.md`. That correction changed concrete module collection to post-order traversal and fixed `optional-typescript`, but the resulting line order still depends on the position and traversal order of dependency operations. Features are appended directly during recursion, so modules and features do not yet use one explicit landscape-defined ordering mechanism.

`src/main/java/com/seed4j/cli/command/infrastructure/primary/ApplyModuleDependencyPlanner.java` currently combines recursive discovery, duplicate suppression, status calculation, and plan-line construction in `DependencyPlanningProgress`.

`src/main/java/com/seed4j/cli/command/infrastructure/primary/ApplyModuleSubCommand.java` currently passes only `modules.resources()` and project history to the planner, even though `Seed4JModulesApplicationService` also exposes the public `landscape()` API.

`src/test/java/com/seed4j/cli/command/infrastructure/primary/Seed4JCommandsFactoryTest.java` contains the stable CLI behavior tests for `apply ... --plan`, including the existing `optional-typescript` landscape-order characterization.

## Desired End State

The planner performs three explicit phases: discover the unique dependency closure; select and order that closure by landscape level and slug; convert ordered dependencies to plan lines using unchanged module and feature status calculations.

Planning `optional-typescript` continues to produce:

```text
module:init
module:prettier
module:typescript
```

Planning `seed4j-extension` produces:

```text
feature:java-build-tool
module:java-base
module:spring-boot
```

The requested module remains outside the plan, and `gradle-java` and `maven-java` remain candidates rather than being added to the dependency closure.

## Milestones

### Milestone 1 - Characterize the mixed dependency order

#### Goal

Capture the public CLI behavior for a transitive closure containing both an abstract feature and concrete modules, including the no-mutation guarantee of plan mode.

#### Changes

- Edit `src/test/java/com/seed4j/cli/command/infrastructure/primary/Seed4JCommandsFactoryTest.java` to add `shouldPlanTransitiveModuleAndFeatureDependenciesInLandscapeOrder`.
- Execute `apply seed4j-extension --plan` against a fresh project fixture.
- Assert the exact dependency section and unchanged commits and project history.

#### Validation

- Command: `./mvnw -Dtest=Seed4JCommandsFactoryTest\$ApplyModule#shouldPlanTransitiveModuleAndFeatureDependenciesInLandscapeOrder test`
- Expected result: passes as a characterization of the current post-order implementation.
- Command: `./mvnw -Dtest=Seed4JCommandsFactoryTest\$ApplyModule#shouldPlanTransitiveModuleDependenciesInLandscapeOrder test`
- Expected result: passes and preserves the previous ordering fix.

#### Acceptance Criteria

- The test observes the CLI command path rather than planner internals.
- Output lists `feature:java-build-tool`, `module:java-base`, then `module:spring-boot`.
- Plan mode creates no commits and no project-history actions.

### Milestone 2 - Separate discovery from landscape ordering

#### Goal

Make dependency ordering explicitly derive from `Seed4JLandscape`, independent of recursive operation position.

#### Changes

- Edit `ApplyModuleDependencyPlanner.java` so `plan(...)` receives `Seed4JLandscape`.
- Replace `DependencyPlanningProgress` with discovery progress containing only `Set<Seed4JLandscapeDependency> dependencies` and `Set<String> visitedModules`.
- Make recursion discover direct and transitive dependencies without constructing lines or assigning positions.
- Index the closure by `Seed4JSlug`, traverse landscape levels in existing order, sort matching slugs within each level, and convert the ordered dependencies to plan lines afterward.
- Rename helpers so `discoverDependencies`, `discoverDependency`, `orderedDependencies`, and `toPlanLine` express the distinct phases.
- Keep module and feature status calculations unchanged.
- Edit `ApplyModuleSubCommand.java` to obtain both `Seed4JModulesResources` and `Seed4JLandscape` from the application service and pass both to the planner.

#### Validation

- Command: `./mvnw -Dtest=Seed4JCommandsFactoryTest\$ApplyModule#shouldPlanTransitiveModuleAndFeatureDependenciesInLandscapeOrder test`
- Expected result: passes after the refactor.
- Command: `./mvnw -Dtest=Seed4JCommandsFactoryTest\$ApplyModule#shouldPlanTransitiveModuleDependenciesInLandscapeOrder test`
- Expected result: passes after the refactor.

#### Acceptance Criteria

- Recursive code only discovers dependencies and tracks visited modules.
- Duplicate dependencies such as `feature:java-build-tool` occur once.
- Modules and features are ordered by the same landscape-level pipeline.
- Existing status text and feature candidate behavior remain unchanged.

### Milestone 3 - Regression and formatting validation

#### Goal

Confirm the refactor preserves all command behavior and repository test expectations.

#### Changes

- Refactor only while the relevant suite remains green.
- Format only the files changed by this work and this ExecPlan if needed.

#### Validation

- Command: `./mvnw -Dtest=Seed4JCommandsFactoryTest test`
- Expected result: all command factory tests pass.
- Command: `./mvnw test`
- Expected result: the default agent-side test gate passes.
- Command: `npx prettier --check <changed files and ExecPlan>`
- Expected result: all task-scoped files conform to Prettier.

#### Acceptance Criteria

- Focused and full Maven tests pass.
- Task-scoped formatting passes.
- Validation outcomes are recorded below.

### Milestone 4 - Finalize the living plan

#### Goal

Leave a handoff-safe record of implementation choices, validation evidence, risks, and lessons.

#### Changes

- Complete all living sections with actual outcomes.
- Move this file from `shared` to `shared/done` only after all requested validation succeeds.

#### Validation

- Command: `git status --short`
- Expected result: only intended source, test, and ExecPlan changes are present alongside any explicitly noted pre-existing untracked context.

#### Acceptance Criteria

- Every milestone is complete.
- The finalized ExecPlan is under `shared/done`.

## Progress

- [x] ExecPlan created and prior fix plan reviewed.
- [x] Milestone 1 started.
- [x] Milestone 1 completed.
- [x] Milestone 2 started.
- [x] Milestone 2 completed.
- [x] Milestone 3 started.
- [x] Milestone 3 completed.
- [x] Milestone 4 started.
- [x] Milestone 4 completed.

## Decisions

- Decision: Treat the new CLI test as a green characterization before refactoring.
  Rationale: The current post-order implementation can already emit the expected `seed4j-extension` output, so forcing an implementation-detail failure would violate behavior-focused TDD.
  Date/Author: 2026-07-20 / Codex

- Decision: Use `Seed4JLandscape.levels()` as the sole ordering authority after closure discovery.
  Rationale: This public API represents Seed4J's dependency levels and includes both module and feature slugs, eliminating traversal-order coupling without changing the upstream library.
  Date/Author: 2026-07-20 / Codex

- Decision: Keep immutable discovery progress and fold dependency transformations through it.
  Rationale: This follows repository guidance against mutable accumulators while limiting recursive state to the closure and visited module slugs.
  Date/Author: 2026-07-20 / Codex

## Risks and Mitigations

- Risk: A dependency discovered from resources might be absent from landscape levels and silently disappear from the rendered plan.
  Mitigation: Validate current public CLI scenarios and inspect the landscape contract; record any mismatch as an architecture gate before broadening behavior.

- Risk: A feature and module could theoretically share an equal `Seed4JSlug`, causing a map collision.
  Mitigation: Use the landscape's typed slug equality contract as designed by Seed4J and allow collection failure to expose invalid resource data rather than inventing an ambiguous order.

- Risk: Multiple declarations of one feature could render duplicate lines.
  Mitigation: Store discovered dependencies in a set before indexing and validate `feature:java-build-tool` appears once in the exact expected dependency section.

- Risk: Repository-wide generated or temporary content may already be untracked.
  Mitigation: Preserve existing files, use task-scoped diffs, and do not stage or remove unrelated content.

- Risk: Existing behavior tests may encode the former alphabetical or traversal order for dependency lines.
  Mitigation: Update only order expectations that demonstrably conflict with landscape levels, preserve their status assertions, and rerun the complete command factory suite.

## Validation Strategy

Run the two focused CLI behavior tests before and after the refactor, then the whole `Seed4JCommandsFactoryTest`, then `./mvnw test`. Run Prettier only against changed files and this ExecPlan. Do not run `./mvnw clean verify`; the user can run that complete gate locally if needed.

Validation results will be appended here as commands run.

- Before refactoring, `./mvnw -Dtest=Seed4JCommandsFactoryTest\$ApplyModule#shouldPlanTransitiveModuleAndFeatureDependenciesInLandscapeOrder test` passed with 1 test and rendered the expected feature/module order without mutations.
- Before refactoring, `./mvnw -Dtest=Seed4JCommandsFactoryTest\$ApplyModule#shouldPlanTransitiveModuleDependenciesInLandscapeOrder test` passed with 1 test and preserved `init`, `prettier`, `typescript` order.
- After refactoring, `./mvnw -Dtest=Seed4JCommandsFactoryTest\$ApplyModule#shouldPlanTransitiveModuleAndFeatureDependenciesInLandscapeOrder test` passed with 1 test and the expected mixed dependency order.
- After refactoring, `./mvnw -Dtest=Seed4JCommandsFactoryTest\$ApplyModule#shouldPlanTransitiveModuleDependenciesInLandscapeOrder test` passed with 1 test and the expected concrete-module order.
- The first post-refactor `./mvnw -Dtest=Seed4JCommandsFactoryTest test` run failed 1 of 40 tests because `shouldPlanFeatureDependencyStatuses` expected alphabetical feature order. Actual output preserved both statuses but correctly placed `feature:java-build-tool` before the later landscape-level `feature:code-coverage-java`; the expectation was updated to the landscape order.
- After updating that landscape-order expectation, `./mvnw -Dtest=Seed4JCommandsFactoryTest test` passed with 40 tests, 0 failures, and 0 errors.
- `./mvnw test` passed with 484 tests, 0 failures, and 0 errors.
- `npx prettier --check` passed for `ApplyModuleDependencyPlanner.java`, `ApplyModuleSubCommand.java`, `Seed4JCommandsFactoryTest.java`, and this ExecPlan.
- `git diff --check` passed with no whitespace errors.
- `./mvnw clean verify` was intentionally not run, per repository and task instructions.

## Rollout and Recovery

This is an internal planner refactor with unchanged CLI syntax and text format. Rollout requires no configuration or migration. If validation reveals a regression, revert the planner and subcommand edits while retaining the behavior tests and this plan's evidence, then restore the last green state before trying a narrower refactor.

## Lessons Learned

- The preceding fix established correct order for one concrete-module graph using post-order traversal, but that mechanism is still coupled to recursive operation order and does not make the landscape the explicit authority.
- `Seed4JModulesApplicationService` in the current `seed4j` 2.2.0 dependency already exposes both `resources()` and `landscape()`, so no dependency publication is required.
- The mixed feature/module scenario is green before the refactor, confirming it is a characterization test rather than a regression-producing RED cycle.
- The broader command suite exposed one prior test coupled to alphabetical feature order. Landscape ordering intentionally changes that line order while leaving applied and pending-choice status calculations unchanged.
- Separating the phases removed the need to carry rendered lines or token-based duplicate state through recursion; discovery now owns only dependency closure and visited-module cycle protection.
- A single landscape traversal can order abstract features and concrete modules because `Seed4JLandscapeLevel.slugs()` exposes both kinds through the common `Seed4JSlug` type.
