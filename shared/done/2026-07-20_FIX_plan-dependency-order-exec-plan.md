# Fix plan dependency order

This ExecPlan is a living document. Keep `Progress`, `Decisions`, `Risks`, and `Lessons Learned` up to date as work advances.

## Purpose / Big Picture

`seed4j apply <module> --plan` prints what module dependencies would be needed before applying the requested module. For modules with transitive dependencies, the dependency lines must match the order Seed4J uses when it applies modules, so an LLM or human can read the plan as an executable sequence. The observable bug is `seed4j apply optional-typescript --plan`: it currently lists `typescript`, `init`, `prettier`, but the expected order is `init`, `prettier`, `typescript`.

## Scope

In scope: change the dependency plan ordering for concrete `module:<slug>` lines, add a behavior test through the CLI command factory path, and validate with focused and default Maven test commands.

Out of scope: adding new CLI options, changing plan line text, automatically resolving `feature:<slug>` dependencies to module candidates, changing module application behavior, or running `./mvnw clean verify` automatically.

Safety boundary: This task is limited to authorized, defensive maintenance of this repository. Do not provide offensive guidance or policy-bypassing instructions.

## Definitions

`--plan` is the CLI flag that prints resolved parameters and dependency status without applying changes.

`Dependency plan` is the rendered section listing direct and transitive dependencies of the requested module.

`module:<slug>` is a concrete module dependency line, such as `module:init`.

`feature:<slug>` is an abstract capability dependency line. It may be satisfied by one of several modules and should keep the existing `satisfied by ...` or `pending choice ...` semantics.

`Landscape order` means the dependency-before-dependent order used by Seed4J's module landscape sorting logic when modules are applied.

## Existing Context

`src/main/java/com/seed4j/cli/command/infrastructure/primary/ApplyModuleSubCommand.java` handles `apply <module>` and calls `ApplyModuleDependencyPlanner` when `--plan` is present.

`src/main/java/com/seed4j/cli/command/infrastructure/primary/ApplyModuleDependencyPlanner.java` recursively walks dependencies and currently sorts each dependency set by textual token. That local token ordering can put a dependent module before its transitive prerequisites.

`src/test/java/com/seed4j/cli/command/infrastructure/primary/Seed4JCommandsFactoryTest.java` already contains integration-level behavior tests for `apply ... --plan`, including module and feature dependency status rendering.

## Desired End State

Planning `optional-typescript` prints concrete module dependencies in application order:

```text
○ module:init - pending
○ module:prettier - pending
○ module:typescript - pending
```

Feature dependencies retain their current status text and are not converted into concrete module choices. Existing plan tests continue to pass.

## Milestones

### Milestone 1 - Capture the behavior

#### Goal

Add a CLI-level test that reproduces the wrong order for transitive concrete module dependencies.

#### Changes

- [ ] Edit `src/test/java/com/seed4j/cli/command/infrastructure/primary/Seed4JCommandsFactoryTest.java` to add `shouldPlanTransitiveModuleDependenciesInLandscapeOrder`.
- [ ] Assert the rendered `Dependency plan` for `optional-typescript` contains `module:init`, `module:prettier`, then `module:typescript`.

#### Validation

- [ ] Command: `./mvnw -Dtest=Seed4JCommandsFactoryTest\\$ApplyModule#shouldPlanTransitiveModuleDependenciesInLandscapeOrder test`
- [ ] Expected result: fails because current output orders concrete module dependencies incorrectly.

#### Acceptance Criteria

- [ ] The test observes the CLI output path, not internal planner details.
- [ ] The failing assertion shows the current order differs from desired landscape order.

### Milestone 2 - Order concrete module closure

#### Goal

Update planning so collected concrete module dependencies are rendered in dependency-before-dependent order while preserving feature dependency behavior.

#### Changes

- [ ] Edit `src/main/java/com/seed4j/cli/command/infrastructure/primary/ApplyModuleDependencyPlanner.java`.
- [ ] Collect the concrete module dependency closure without duplicates.
- [ ] Order concrete module dependency lines so transitive prerequisites appear before dependents.
- [ ] Preserve feature dependency lines and statuses.
- [ ] Adjust `ApplyModuleSubCommand` only if the planner needs an additional stable caller input.

#### Validation

- [ ] Command: `./mvnw -Dtest=Seed4JCommandsFactoryTest\\$ApplyModule#shouldPlanTransitiveModuleDependenciesInLandscapeOrder test`
- [ ] Expected result: passes.

#### Acceptance Criteria

- [ ] `optional-typescript --plan` prints `init`, `prettier`, `typescript` in that order.
- [ ] The requested module itself is not listed in its dependency plan.
- [ ] Feature dependency lines keep their existing rendering semantics.

### Milestone 3 - Regression validation

#### Goal

Confirm the command factory plan behavior still works across existing scenarios.

#### Changes

- [ ] Refactor only if tests are green and the code can be made clearer without broadening scope.

#### Validation

- [ ] Command: `./mvnw -Dtest=Seed4JCommandsFactoryTest test`
- [ ] Expected result: passes.
- [ ] Command: `./mvnw test`
- [ ] Expected result: passes.

#### Acceptance Criteria

- [ ] Focused command tests pass.
- [ ] Default agent-side Maven test gate passes.

## Progress

- [x] ExecPlan created
- [x] Milestone 1 started
- [x] Milestone 1 completed
- [x] Milestone 2 started
- [x] Milestone 2 completed
- [x] Milestone 3 started
- [x] Milestone 3 completed

## Decisions

- Decision: Use `Seed4JCommandsFactoryTest` as the test home.
  Rationale: The bug is observable through CLI `--plan` output, and this class already covers the stable command factory path without mirroring planner internals.
  Date/Author: 2026-07-20 / Codex

- Decision: Preserve the planner's existing public inputs.
  Rationale: The current resources collection contains enough module dependency information to produce dependency-before-dependent order; no new CLI or application service API is needed.
  Date/Author: 2026-07-20 / Codex

- Decision: Use post-order traversal for concrete module dependencies.
  Rationale: Appending a module dependency after its own dependencies puts transitive prerequisites before dependents while leaving feature status resolution unchanged.
  Date/Author: 2026-07-20 / Codex

## Risks and Mitigations

- Risk: Reordering feature dependencies could unintentionally change existing CLI output.
  Mitigation: Keep feature dependency handling separate and run existing `Seed4JCommandsFactoryTest` scenarios.

- Risk: A custom ordering implementation could diverge from Seed4J's application dependency order.
  Mitigation: Prefer dependency-before-dependent traversal over alphabetical token sorting for concrete module closure, and validate with the known `optional-typescript` case.

- Risk: Repository-wide formatting check can be noisy because unrelated files may already be out of Prettier format.
  Mitigation: Run the repository check to observe the current state, then run Prettier check on the files touched by this task.

## Validation Strategy

1. Run the focused RED command for the new behavior test.
2. Run the same focused command after implementation.
3. Run `./mvnw -Dtest=Seed4JCommandsFactoryTest test`.
4. Run `./mvnw test` as the default agent-side validation gate.
5. Ask the user to run `./mvnw clean verify` if the complete local gate is needed.

Validation results:

- `./mvnw -Dtest=Seed4JCommandsFactoryTest\\$ApplyModule#shouldPlanTransitiveModuleDependenciesInLandscapeOrder test` failed in RED with the observed current order `module:typescript`, `module:init`, `module:prettier`.
- `./mvnw -Dtest=Seed4JCommandsFactoryTest\\$ApplyModule#shouldPlanTransitiveModuleDependenciesInLandscapeOrder test` passed after implementation.
- `./mvnw -Dtest=Seed4JCommandsFactoryTest test` passed with 39 tests, 0 failures.
- `./mvnw test` passed with 483 tests, 0 failures.
- `npm run prettier:check` failed on 8 pre-existing files not touched by this task: `_temporary/ai_agent/seed4j-cli-ai-context/shared/done/2026-06-22_FEATURE_publish-seed4j-cli-to-npm-exec-plan.md`, `src/main/java/com/seed4j/cli/bootstrap/infrastructure/secondary/JavaProcessChildLauncher.java`, `src/main/java/com/seed4j/cli/command/infrastructure/primary/BashCompletionScriptGenerator.java`, `src/main/java/com/seed4j/cli/command/infrastructure/primary/Seed4JCommandsFactory.java`, `src/test/java/com/mycompany/seed4j/extension/runtime/main/apply/RuntimeExtensionCommonSourceNodePackagesVersionsReader.java`, `src/test/java/com/seed4j/cli/bootstrap/infrastructure/primary/JavaRuntimeExtensionInstallerTest.java`, `src/test/java/com/seed4j/cli/command/infrastructure/primary/ExtensionRuntimeCommandsTest.java`, and `src/test/java/com/seed4j/cli/HexagonalArchTest.java`.
- `npx prettier --check src/main/java/com/seed4j/cli/command/infrastructure/primary/ApplyModuleDependencyPlanner.java src/test/java/com/seed4j/cli/command/infrastructure/primary/Seed4JCommandsFactoryTest.java _temporary/ai_agent/seed4j-cli-ai-context/shared/2026-07-20_FIX_plan-dependency-order-exec-plan.md` passed.

## Rollout and Recovery

This is a CLI output ordering change only. If a regression appears, revert the changes to `ApplyModuleDependencyPlanner.java` and the added test, then rerun the focused command factory test.

## Lessons Learned

- The initial selector `Seed4JCommandsFactoryTest#shouldPlanTransitiveModuleDependenciesInLandscapeOrder` matched zero tests because the behavior lives in the nested `ApplyModule` test class. Use `Seed4JCommandsFactoryTest\\$ApplyModule#shouldPlanTransitiveModuleDependenciesInLandscapeOrder` for the focused method.
- The RED output confirmed the concrete current order is `module:typescript`, `module:init`, `module:prettier`.
- The GREEN output for the focused method confirmed the rendered order changed to `module:init`, `module:prettier`, `module:typescript`.
- Repository-wide Prettier currently reports unrelated formatting drift, so task-scoped formatting verification is needed to distinguish this change from existing workspace state.
