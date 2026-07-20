# Block apply when dependencies are missing

This ExecPlan is a living document. Keep `Progress`, `Decisions`, `Risks`, and `Lessons Learned` up to date as work advances.

## Purpose / Big Picture

`seed4j apply <module>` currently generates a module even when its required module or feature dependencies are absent. This change makes normal apply perform the same dependency planning already used by `apply --plan`, reject an incomplete plan with exit code 2 before validating required parameters, and leave the project, Git history, and Seed4J action history unchanged. Users can observe the behavior through a deterministic stderr diagnostic that lists only pending dependencies and tells them how to recover.

Safety boundary: This task is limited to authorized, defensive maintenance of this repository. Do not provide offensive guidance or policy-bypassing instructions.

## Scope

In scope:

- Reuse `ApplyModuleDependencyPlanner` for both normal apply and `--plan`.
- Block direct, transitive, and feature dependencies that remain pending.
- Keep pending dependencies in landscape order and feature candidates in their existing alphabetical order.
- Validate dependency readiness before required module parameters.
- Render the prescribed missing-dependency diagnostic to stderr and return `picocli.CommandLine.ExitCode.USAGE` (2).
- Preserve normal apply behavior when the plan is ready and preserve read-only, zero-exit `--plan` behavior regardless of readiness.
- Update `README.md` and `documentation/Commands.md`.

Out of scope:

- Automatically applying dependencies.
- Post-apply warnings.
- Adding `--strict-dependencies`, a bypass flag, migration, or rollout configuration.
- Changing public Java APIs or making dependency readiness a universal Seed4J generator invariant.

## Definitions

- Dependency plan: the existing primary-adapter model produced by `ApplyModuleDependencyPlanner` from the target module, visible module resources, landscape, and project history.
- Ready plan: a plan whose dependency statuses are all resolved (`already applied` or `satisfied by`).
- Pending line: a plan line whose status is `pending` or `pending choice`.
- Feature dependency: a required capability satisfied when any visible provider module is recorded in project history.
- Landscape order: the deterministic ordering already produced by `ApplyModuleDependencyPlanner.orderedDependencies`.
- Apply history: `ProjectHistory.actions()`, the existing source of truth for applied module slugs.

## Existing Context

The completed plan `_temporary/ai_agent/seed4j-cli-ai-context/shared/done/2026-06-26_FEATURE_apply-plan-dependency-status-exec-plan.md` introduced `ApplyModuleDependencyPlanner`, `ApplyModuleDependencyPlan`, `ApplyModuleDependencyStatus`, and the dependency section of `ApplyModulePlanRenderer`. It established transitive discovery, landscape ordering, history-based module satisfaction, and alphabetical visible feature candidates.

`src/main/java/com/seed4j/cli/command/infrastructure/primary/ApplyModuleSubCommand.java` currently constructs the dependency plan only inside its `--plan` branch. Normal apply merges parameters, validates mandatory parameters, and calls `modules.apply(...)` without checking dependencies.

`src/test/java/com/seed4j/cli/command/infrastructure/primary/Seed4JCommandsFactoryTest.java`, nested class `ApplyModule`, is the integration boundary for CLI exit codes, output, commits, and project action history.

The unfinished `_temporary/ai_agent/seed4j-cli-ai-context/shared/FEATURE_DEPENDENCY_WARNINGS_AND_STRICT_DEPENDENCIES_EXEC_PLAN.md` is superseded by issue #200 and this plan. Its optional warnings, post-apply warnings, module-only strict behavior, and `--strict-dependencies` flag must not be implemented.

## Desired End State

Normal apply computes one `ApplyModuleDependencyPlan` before branching. `--plan` renders that plan and returns zero without applying or blocking. Normal apply asks the plan whether it is ready before validating mandatory parameters. If not ready, a dedicated primary renderer writes exactly the pending lines to stderr, returns exit code 2, and never calls `modules.apply`. If ready, existing required-parameter validation and application behavior continue unchanged.

The blocked diagnostic has this shape:

    Cannot apply module: <module>

    Missing required dependencies:

    ○ module:<slug> - pending
    ○ feature:<feature> - pending choice: <candidate>, <candidate>

    Next action: apply every pending module and one module from each pending choice, then retry this module.
    No changes were applied.

Resolved dependency lines never appear in this diagnostic.

## Milestones

### Milestone 1 - Establish the behavior contract

#### Goal

Create this living plan and prove that a missing direct dependency blocks normal apply without side effects.

#### Changes

- [x] Create this ExecPlan and reference the completed dependency-status plan.
- [x] Add one integration behavior in `Seed4JCommandsFactoryTest.ApplyModule` for a missing direct module dependency.
- [x] Add the smallest primary-adapter readiness and rendering behavior needed to pass it.

#### Validation

- [x] Command: `./mvnw -Dtest='Seed4JCommandsFactoryTest$ApplyModule' test`
- [x] Expected result: the new test first failed because required-parameter validation ran before dependency readiness; after the production change, 28 focused tests passed with the prescribed exit code, diagnostic, and unchanged commits/history.

#### Acceptance Criteria

- [x] A direct pending dependency blocks before generation.
- [x] The error contains only the pending direct dependency.

### Milestone 2 - Cover transitive and feature readiness

#### Goal

Prove that the reused plan blocks every unresolved dependency while retaining its existing ordering and satisfaction semantics.

#### Changes

- [x] Add a separate behavior for transitive pending dependencies in landscape order with applied dependencies omitted.
- [x] Add a separate behavior for an unsatisfied feature with all visible providers alphabetically ordered.
- [x] Add a separate behavior showing that an applied visible feature provider permits normal apply.
- [x] Add `ApplyModuleDependencyStatus.satisfied()`, `ApplyModuleDependencyPlan.ready()`, and `pendingLines()` without duplicating status rules in the command.

#### Validation

- [x] Command after each cycle: `./mvnw -Dtest='Seed4JCommandsFactoryTest$ApplyModule' test`
- [x] Expected result: the transitive, missing-feature, and satisfied-feature behaviors were already green under the generic preflight introduced by the first failing cycle; the nested suite progressed from 29 to 31 passing tests without production changes between those behavior additions.

#### Acceptance Criteria

- [x] Direct, transitive, and feature dependencies use the same planner as `--plan`.
- [x] Resolved dependencies are omitted from blocked output.
- [x] A history-recorded visible feature provider satisfies the feature.

### Milestone 3 - Preserve validation precedence and compatible paths

#### Goal

Lock dependency validation ahead of required parameters and preserve modules without dependencies and `--plan`.

#### Changes

- [x] Add a behavior where both dependencies and mandatory parameters are missing and only dependency readiness decides first.
- [x] Add or adjust a behavior proving modules without dependencies apply normally and `--plan` remains zero-exit and read-only with pending dependencies.
- [x] Refactor `ApplyModuleSubCommand` so one dependency plan instance serves both execution modes.

#### Validation

- [x] Command after each cycle: `./mvnw -Dtest='Seed4JCommandsFactoryTest$ApplyModule' test`
- [x] Expected result: 32 nested apply tests pass; the isolated precedence behavior sees dependency stderr without required-option output, while existing init apply and pending `--plan` behaviors remain green.

#### Acceptance Criteria

- [x] Missing dependencies return 2 before `Missing required options` can be produced.
- [x] Ready normal applies and all plan invocations retain existing behavior.

### Milestone 4 - Documentation and repository validation

#### Goal

Document the new CLI contract and complete agent-side validation.

#### Changes

- [x] Update `README.md` with blocking behavior, recovery action, and the `--plan` distinction.
- [x] Update `documentation/Commands.md` with validation precedence and direct-module and feature examples.
- [x] Format supported files with the repository formatter if required.

#### Validation

- [x] Command: `npm run prettier:check`
- [x] Expected result: all touched files pass a direct Prettier check; the repository-wide check reports eight unrelated pre-existing files listed in `Lessons Learned`.
- [x] Command: `./mvnw test`
- [x] Expected result: all 489 tests pass with no failures, errors, or skips.
- [ ] User-run final gate: `./mvnw clean verify`
- [ ] Expected result: user reports exit code 0; otherwise the relevant failure summary is used for follow-up.

#### Acceptance Criteria

- [x] Both documentation files describe the exact contract and examples.
- [x] Focused and full agent-side test suites pass.

## Progress

- [x] Milestone 1 started
- [x] Milestone 1 completed
- [x] Milestone 2 started
- [x] Milestone 2 completed
- [x] Milestone 3 started
- [x] Milestone 3 completed
- [x] Milestone 4 started
- [x] Milestone 4 completed

## Decisions

- Decision: Reuse the exact `ApplyModuleDependencyPlanner` result already used by `--plan` and construct it once in `ApplyModuleSubCommand.call()`.
  Rationale: One plan prevents semantic drift between inspection and enforcement.
  Date/Author: 2026-07-20 / User + Codex

- Decision: Keep blocking and its renderer in `command.infrastructure.primary`.
  Rationale: This is a CLI policy over an existing CLI planning model, not a universal generator invariant.
  Date/Author: 2026-07-20 / User + Codex

- Decision: Dependency validation precedes required-parameter validation.
  Rationale: This precedence was explicitly selected for issue #200 and gives callers the safe prerequisite action first.
  Date/Author: 2026-07-20 / User + Codex

- Decision: The old optional warnings and `--strict-dependencies` plan is superseded.
  Rationale: Issue #200 requires unconditional preflight blocking for normal apply, including feature dependencies, with no warning-only or bypass mode.
  Date/Author: 2026-07-20 / User + Codex

## Risks and Mitigations

- Risk: Computing separate plans could allow `--plan` and normal apply to disagree.
  Mitigation: Construct one plan before the execution-mode branch and pass the same value onward.

- Risk: A renderer could duplicate resolved-versus-pending rules.
  Mitigation: Put status classification in `ApplyModuleDependencyStatus.satisfied()` and selection in `ApplyModuleDependencyPlan.pendingLines()`.

- Risk: Blocking tests could accidentally prove only text while generation still occurs.
  Mitigation: Assert exit code, commits, and project history actions at the CLI integration boundary.

- Risk: Existing successful tests apply dependent modules out of order and will fail under the new contract.
  Mitigation: Treat failures as contract updates only when their setup does not satisfy real prerequisites; preserve each test's original user-visible purpose by applying prerequisites through the CLI.

- Risk: Repository-wide formatting may contain unrelated drift.
  Mitigation: Unrelated files were not modified; every touched file passed a direct Prettier check and the eight existing failures are recorded below.

## Validation Strategy

1. Run `./mvnw -Dtest='Seed4JCommandsFactoryTest$ApplyModule' test` for every RED and GREEN cycle.
2. Maintain a public CLI-path checkpoint at least every two cycles through the same nested integration suite.
3. Run `npm run prettier:check` after code and documentation changes.
4. Run `./mvnw test` as the final agent-side gate.
5. Ask the user to run `./mvnw clean verify` and report its exit code plus a concise relevant failure summary if nonzero.

## Rollout and Recovery

Release through the normal CLI release process; no migration or feature flag is required. If the new preflight causes a regression, revert the normal-apply readiness check and dedicated missing-dependency renderer while preserving the existing `--plan` planner and renderer. This restores the former apply behavior without changing stored project history formats.

## Lessons Learned

- The completed apply-plan work already provides transitive discovery, landscape ordering, visible provider filtering, and history-based satisfaction, so issue #200 should add enforcement rather than a second resolver.
- The previous warnings/strict plan conflicts with the now-approved unconditional blocking contract and must remain unimplemented.
- The first RED failed at the required `--package-name` validation for `angular-core`, directly proving the old precedence. The first GREEN passed all 28 nested apply tests after moving readiness ahead of mandatory parameters.
- The generic readiness model from the first GREEN also satisfied the separately added transitive and feature behaviors on their first runs. No artificial production regression was introduced merely to force additional RED states; each behavior remains independently asserted at the CLI boundary.
- Existing `shouldApplyInitModule...` and `shouldPlan...` integration behaviors already cover the no-dependency and read-only plan compatibility paths; both remained green throughout the new preflight work.
- `./mvnw test` passed all 489 tests. Maven still emits the repository's known dependency-convergence warnings, but they are non-failing under the current Enforcer configuration.
- `npm run prettier:check` reports unrelated existing drift in the completed npm-publish ExecPlan, `JavaProcessChildLauncher.java`, `BashCompletionScriptGenerator.java`, `Seed4JCommandsFactory.java`, `RuntimeExtensionCommonSourceNodePackagesVersionsReader.java`, `JavaRuntimeExtensionInstallerTest.java`, `ExtensionRuntimeCommandsTest.java`, and `HexagonalArchTest.java`. All files touched by this issue pass Prettier directly.
