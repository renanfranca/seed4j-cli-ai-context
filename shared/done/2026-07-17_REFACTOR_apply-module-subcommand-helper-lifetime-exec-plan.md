# Refactor ApplyModuleSubCommand Helper Lifetime

This ExecPlan is a living document. Keep `Progress`, `Decisions`, `Risks`, and `Lessons Learned` up to date as work advances.

## Purpose / Big Picture

This refactor keeps the `seed4j apply <module>` command behavior unchanged while reducing unnecessary object lifetime in `ApplyModuleSubCommand`. Users should observe the same CLI help, `--plan` output, module application behavior, project history behavior, and exit codes before and after the change. The code improvement is internal: plan-only helpers should be created near the `--plan` flow instead of being retained for every command instance.

Safety boundary: This task is limited to authorized, defensive maintenance of this repository. Do not provide offensive guidance or policy-bypassing instructions.

## Scope

In scope:

- Update `src/main/java/com/seed4j/cli/command/infrastructure/primary/ApplyModuleSubCommand.java`.
- Make `KnownModulePropertyCompletionCandidates` a `private static final` field.
- Remove instance fields for `ApplyModuleParameterResolver`, `ApplyModuleDependencyPlanner`, and `ApplyModulePlanRenderer`.
- Extract the `--plan` rendering flow into a private method that instantiates plan helpers near use.
- Resolve `ProjectPath`, `ProjectHistory`, explicit module parameters, historical parameters, and merged parameters once in `call()`.
- Validate unchanged behavior using existing public-path tests.

Out of scope:

- Changing CLI command names, help text, options, output text, exit codes, commits, project history persistence, public APIs, or domain types.
- Adding tests that assert helper instantiation, object lifetime, field presence, or private method structure.
- Running `./mvnw clean verify` automatically.

## Definitions

`ApplyModuleSubCommand` is the picocli primary adapter that builds and executes a dynamic `apply` subcommand for one Seed4J module.

`--plan` is the apply command mode that prints resolved parameters and dependency planning without applying changes.

`ProjectPath` is the domain value object for a user-visible project path.

`ProjectHistory` records previously applied project actions and module properties.

`ModuleParameters` represents module parameter values from CLI options or project history.

Helper lifetime means how long helper objects remain reachable. In this refactor, helpers used only by `--plan` should be local to that flow.

## Existing Context

`src/main/java/com/seed4j/cli/command/infrastructure/primary/ApplyModuleSubCommand.java` currently owns instance fields for `KnownModulePropertyCompletionCandidates`, `ApplyModuleParameterResolver`, `ApplyModuleDependencyPlanner`, and `ApplyModulePlanRenderer`.

`KnownModulePropertyCompletionCandidates` is used while building dynamic options and examples. It has no per-command mutable state and can be shared as a static final singleton.

`ApplyModuleParameterResolver`, `ApplyModuleDependencyPlanner`, and `ApplyModulePlanRenderer` are only needed when `executionMode()` is `ApplyModuleExecutionMode.PLAN`. Keeping them as command instance fields expands their lifetime without changing behavior.

The existing behavior tests in `src/test/java/com/seed4j/cli/command/infrastructure/primary/ApplyModuleSubCommandTest.java` and `src/test/java/com/seed4j/cli/command/infrastructure/primary/Seed4JCommandsFactoryTest.java` cover the stable command behavior, including plan-mode behavior.

## Desired End State

`ApplyModuleSubCommand.call()` resolves the project path, project history, explicit parameters, history parameters, and merged parameters once. It delegates plan output to a private method when `--plan` is active. That private method creates `ApplyModuleParameterResolver`, `ApplyModuleDependencyPlanner`, and `ApplyModulePlanRenderer` locally and prints the same plan output as before.

The command remains behaviorally identical from the CLI and test perspective.

## Milestones

### Milestone 1 - Establish Plan and Baseline Contract

#### Goal

Create this living ExecPlan and validate the current public-path behavior before refactoring.

#### Changes

- [x] Add `_temporary/ai_agent/seed4j-cli-ai-context/shared/2026-07-17_REFACTOR_apply-module-subcommand-helper-lifetime-exec-plan.md`.
- [x] Run focused tests before production edits.

#### Validation

- [x] Command: `./mvnw -Dtest=ApplyModuleSubCommandTest,Seed4JCommandsFactoryTest test`
- [x] Expected result: tests pass before the internal refactor.

#### Acceptance Criteria

- [x] The ExecPlan exists in `_temporary/`.
- [x] Existing behavior tests define the refactor contract.

### Milestone 2 - Refactor Helper Lifetime

#### Goal

Apply the internal refactor without changing public behavior.

#### Changes

- [x] Edit `src/main/java/com/seed4j/cli/command/infrastructure/primary/ApplyModuleSubCommand.java`.
- [x] Convert `KnownModulePropertyCompletionCandidates` to a `private static final` field.
- [x] Remove instance fields for plan-only helpers.
- [x] Extract plan-mode rendering into a private method.
- [x] Resolve `ProjectPath`, `ProjectHistory`, and `ModuleParameters` once in `call()`.

#### Validation

- [x] Command: `./mvnw -Dtest=ApplyModuleSubCommandTest,Seed4JCommandsFactoryTest test`
- [x] Expected result: focused behavior tests pass after the refactor.

#### Acceptance Criteria

- [x] No CLI output or behavior changes are introduced.
- [x] Plan-only helpers are instantiated near the plan flow.
- [x] No new implementation-detail tests are added.

### Milestone 3 - Agent-Side Validation

#### Goal

Run the standard agent-side validation gate and inspect the final diff.

#### Changes

- [x] Update this ExecPlan with validation results, decisions, risks, and lessons learned.
- [x] Confirm `_temporary/` remains untracked and is not part of tracked source changes.

#### Validation

- [x] Command: `./mvnw test`
- [x] Expected result: full JUnit suite passes.
- [x] Command: `git status --short`
- [x] Expected result: tracked source changes are limited to the intended refactor; `_temporary/` remains untracked.

#### Acceptance Criteria

- [x] The standard test suite passes.
- [x] The final diff is scoped to `ApplyModuleSubCommand.java` plus the untracked ExecPlan.

## Progress

- [x] Milestone 1 started.
- [x] ExecPlan created.
- [x] Baseline focused tests completed.
- [x] Milestone 1 completed.
- [x] Milestone 2 started.
- [x] Production refactor completed.
- [x] Focused post-refactor tests completed.
- [x] Milestone 2 completed.
- [x] Milestone 3 started.
- [x] Full `./mvnw test` completed.
- [x] Final status inspected.
- [x] Milestone 3 completed.

## Decisions

- Decision: Do not add helper-lifetime or field-structure tests.
  Rationale: The requested change is an internal refactor. Testing helper lifetime would assert implementation topology rather than observable command behavior, contrary to repository testing guidance.
  Date/Author: 2026-07-17 / Codex

- Decision: Use existing `ApplyModuleSubCommandTest` and `Seed4JCommandsFactoryTest` as the focused behavior contract.
  Rationale: These tests exercise the public command surface and plan behavior without coupling to private helpers.
  Date/Author: 2026-07-17 / Codex

- Decision: Format only `ApplyModuleSubCommand.java` after repository-wide Prettier reported unrelated existing formatting issues.
  Rationale: The edited Java file must follow repository formatting, while unrelated files should not be churned as part of this scoped refactor.
  Date/Author: 2026-07-17 / Codex

## Risks and Mitigations

- Risk: Accidentally changing `--plan` output while extracting a method.
  Mitigation: Run focused command tests before and after the refactor.

- Risk: Re-reading project history in plan mode after the refactor.
  Mitigation: Resolve `ProjectHistory` once in `call()` and pass it to the plan method.

- Risk: Accidentally tracking `_temporary/` content.
  Mitigation: Inspect `git status --short` before final response and report that `_temporary/` remains untracked.

## Validation Strategy

1. Run `./mvnw -Dtest=ApplyModuleSubCommandTest,Seed4JCommandsFactoryTest test` before production edits to establish the current behavior contract.
2. Run the same focused tests after the refactor.
3. Run `./mvnw test` as the standard agent-side validation gate.
4. Inspect `git status --short` to confirm scope and untracked `_temporary/` state.

## Rollout and Recovery

This is an internal refactor with no configuration or migration. Rollout is the normal repository change flow. Recovery is to revert the `ApplyModuleSubCommand.java` edit if behavior tests fail or if review finds the helper lifetime change unclear.

Do not run `./mvnw clean verify` automatically. The user can run it locally as the final complete gate when needed.

## Lessons Learned

- Baseline focused test command completed successfully before production edits: 39 tests run, 0 failures, 0 errors, 0 skipped.
- Post-refactor focused test command completed successfully: 39 tests run, 0 failures, 0 errors, 0 skipped.
- Repository-wide `npm run prettier:check` currently fails on multiple pre-existing files outside this refactor, and also initially marked the edited file. Running `npx prettier --write src/main/java/com/seed4j/cli/command/infrastructure/primary/ApplyModuleSubCommand.java` fixed the edited file, and `npx prettier --check src/main/java/com/seed4j/cli/command/infrastructure/primary/ApplyModuleSubCommand.java` passed.
- Final focused test command after formatting completed successfully: 39 tests run, 0 failures, 0 errors, 0 skipped.
- Final `./mvnw test` completed successfully: 482 tests run, 0 failures, 0 errors, 0 skipped.
- Final `git status --short` shows one tracked source modification, `src/main/java/com/seed4j/cli/command/infrastructure/primary/ApplyModuleSubCommand.java`, and untracked `_temporary/`.
