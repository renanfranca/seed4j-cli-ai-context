# Enhance apply plan dependency status

This ExecPlan is a living document. Keep `Progress`, `Decisions`, `Risks`, and `Lessons Learned` up to date as work advances.

## Purpose / Big Picture

`seed4j apply <module> --plan` currently explains parameter values but does not show whether module dependencies are already present or still need attention. This change adds a `Dependency plan` section before `Resolved parameters` so a human or LLM caller can inspect dependency readiness without applying files, creating commits, or writing project history. The behavior is observable by running `seed4j apply <module> --plan` on projects with and without applied dependency modules.

Safety boundary: This task is limited to authorized, defensive maintenance of this repository. Do not provide offensive guidance or policy-bypassing instructions.

## Scope

In scope:

- Render dependency status in `seed4j apply <module> --plan`.
- Use the current project's history to identify already applied modules.
- Use visible module resources to inspect direct module dependencies, feature dependencies, and feature candidates.
- Preserve read-only planning behavior.
- Update README and `documentation/Commands.md`.

Out of scope:

- Changing module application behavior.
- Writing project history during planning.
- Adding MCP planning workflows.
- Exposing hidden storage layout as domain concepts.

## Definitions

- Apply plan: the text output produced by `seed4j apply <module> --plan`.
- Direct module dependency: a dependency declared as a module slug in a module resource organization.
- Transitive dependency: a dependency found by following visible module dependencies from the requested module.
- Feature dependency: a dependency declared as a feature slug in a module resource organization.
- Feature candidate: a visible module whose organization provides a feature that can satisfy a feature dependency.
- Applied module: a module recorded in project history as already applied to the project.
- Pending choice: an unsatisfied feature dependency where one of several visible candidate modules may satisfy the dependency.

## Existing Context

The CLI primary adapter lives under `src/main/java/com/seed4j/cli/command/infrastructure/primary`. `ApplyModuleSubCommand` builds per-module picocli commands, resolves CLI and history parameters, and calls `ApplyModulePlanRenderer` when `--plan` is selected. `ApplyModulePlanRenderer` currently prints `Plan for module`, `Project path`, `Resolved parameters`, optional missing required parameters, and `No changes were applied.`.

The visible module catalog is available through `Seed4JModulesApplicationService.resources()`. `ListModulesCommand` already reads `Seed4JModuleResource.organization().dependencies()` and renders module and feature dependency tokens for `seed4j list`.

Integration tests for CLI behavior are in `src/test/java/com/seed4j/cli/command/infrastructure/primary/Seed4JCommandsFactoryTest.java`.

## Desired End State

`seed4j apply <module> --plan` prints a `Dependency plan` section before `Resolved parameters`. Direct module dependencies show `already applied` when the dependency module is in project history, otherwise `pending`. Feature dependencies show `satisfied by <module>` when an applied module provides the feature, otherwise `pending choice` with sorted visible candidate modules. The plan stays read-only.

## Milestones

### Milestone 1 - Establish Living Plan

#### Goal

Create this ExecPlan and record the intended TDD and validation workflow.

#### Changes

- [x] Create `/home/renanfranca/projects/seed4j-cli/_temporary/ai_agent/seed4j-cli-ai-context/shared/2026-06-26_FEATURE_apply-plan-dependency-status-exec-plan.md`.

#### Validation

- [x] Command: `test -f /home/renanfranca/projects/seed4j-cli/_temporary/ai_agent/seed4j-cli-ai-context/shared/2026-06-26_FEATURE_apply-plan-dependency-status-exec-plan.md`
- [x] Expected result: file exists.

#### Acceptance Criteria

- [x] The ExecPlan is self-contained and lists milestones, validation, risks, and rollout guidance.

### Milestone 2 - Direct Module Dependency Status

#### Goal

Show direct module dependencies in the apply plan, marking already applied dependencies separately from pending dependencies.

#### Changes

- [x] Add a behavior test in `Seed4JCommandsFactoryTest` for direct module dependencies.
- [x] Add primary-adapter planning types as needed under `src/main/java/com/seed4j/cli/command/infrastructure/primary`.
- [x] Pass project history and visible resources into `ApplyModulePlanRenderer`.

#### Validation

- [x] Command: `./mvnw -Dtest=Seed4JCommandsFactoryTest test`
- [x] Expected result: focused CLI test suite passes.

#### Acceptance Criteria

- [x] A plan for a module with direct dependencies renders `module:<slug> - already applied` for applied dependencies.
- [x] The same plan renders `module:<slug> - pending` for unapplied dependencies.
- [x] Planning remains read-only.

### Milestone 3 - Feature Dependency Satisfaction and Choices

#### Goal

Show feature dependency status in the apply plan, including applied satisfiers and sorted pending candidates.

#### Changes

- [x] Add a behavior test in `Seed4JCommandsFactoryTest` for feature dependency satisfaction and pending choices.
- [x] Extend the primary-adapter planning model to discover visible feature candidates.
- [x] Render `feature:<slug> - satisfied by <module>` and `feature:<slug> - pending choice: <candidate>, <candidate>`.

#### Validation

- [x] Command: `./mvnw -Dtest=Seed4JCommandsFactoryTest test`
- [x] Expected result: focused CLI test suite passes.

#### Acceptance Criteria

- [x] Applied feature providers satisfy matching feature dependencies.
- [x] Unsatisfied feature dependencies list visible candidates in sorted order.
- [x] Planning remains read-only.

### Milestone 4 - Documentation and Formatting

#### Goal

Document the expanded apply plan output and run focused formatting validation.

#### Changes

- [x] Update `README.md`.
- [x] Update `documentation/Commands.md`.
- [x] Format changed files if needed.

#### Validation

- [x] Command: `npx prettier --check src/main/java/com/seed4j/cli/command/infrastructure/primary/ApplyModuleDependencyPlan.java src/main/java/com/seed4j/cli/command/infrastructure/primary/ApplyModuleDependencyPlanLine.java src/main/java/com/seed4j/cli/command/infrastructure/primary/ApplyModuleDependencyPlanner.java src/main/java/com/seed4j/cli/command/infrastructure/primary/ApplyModulePlanRenderer.java src/main/java/com/seed4j/cli/command/infrastructure/primary/ApplyModuleSubCommand.java src/test/java/com/seed4j/cli/command/infrastructure/primary/Seed4JCommandsFactoryTest.java README.md documentation/Commands.md`
- [x] Expected result: changed files pass formatting check.
- [x] Command: `npm run prettier:check`
- [x] Expected result: repository-wide formatting check still reports unrelated pre-existing formatting issues in `.mvn/settings-no-mirror.xml`, `src/main/java/com/seed4j/cli/bootstrap/infrastructure/secondary/FileSystemRuntimeExtensionArtifactsRepository.java`, and `src/test/java/com/seed4j/cli/command/infrastructure/primary/BashCompletionScriptGeneratorTest.java`.
- [x] Command: `./mvnw -Dtest=Seed4JCommandsFactoryTest test`
- [x] Expected result: focused CLI test suite passes after docs changes.

#### Acceptance Criteria

- [x] Documentation includes dependency status examples.
- [x] Focused tests and formatting validation pass for changed files; repository-wide formatting has unrelated existing drift.

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

- Decision: Keep dependency planning in `command/infrastructure/primary`.
  Rationale: The change is CLI inspection output over already exposed module resources and project history, not new domain behavior or secondary storage logic.
  Date/Author: 2026-06-26 / Codex

- Decision: Use public CLI integration tests as the first behavior contract.
  Rationale: The user-visible output and read-only guarantee are best observed through `seed4j apply <module> --plan`.
  Date/Author: 2026-06-26 / Codex

- Decision: Render dependency rows without blank lines between rows.
  Rationale: This keeps the new section compact and machine-friendly while preserving blank-line separation between sections.
  Date/Author: 2026-06-26 / Codex

- Decision: Follow visible module dependencies transitively and stop on already visited module slugs.
  Rationale: The issue plan treated dependency chain as transitive; a visited set makes traversal deterministic and cycle-safe without exposing hidden module storage details.
  Date/Author: 2026-06-26 / Codex

## Risks and Mitigations

- Risk: Feature provider APIs in the Seed4J dependency may be less obvious than dependency APIs.
  Mitigation: Inspect the compiled API with local tooling before choosing production types, and keep tests at the CLI boundary.

- Risk: Planning could accidentally call `modules.apply`.
  Mitigation: Assert no commits and no history action changes in behavior tests.

- Risk: Output ordering may be unstable.
  Mitigation: Sort dependency and candidate slugs before rendering.

- Risk: Repository-wide formatting validation can fail on unrelated files.
  Mitigation: Format and check touched files directly, record unrelated files reported by the full check, and avoid changing them in this issue.

## Validation Strategy

1. Run `./mvnw -Dtest=Seed4JCommandsFactoryTest test` after each behavior cycle.
2. Run `npm run prettier:check` after documentation and formatting changes.
3. Ask the user to run `./mvnw clean verify` locally as the final repository gate.

## Rollout and Recovery

This is a CLI output enhancement for `--plan`. Rollout is the normal CLI release path. If a regression appears, revert the changed primary adapter and documentation files; module application behavior should be unaffected because the implementation is only used in plan mode.

## Lessons Learned

- The existing `seed4j list` command is the local reference for reading and displaying module and feature dependency tokens from visible module resources.
- The focused CLI suite produces very large captured output because list and completion tests print full command catalogs; failure summaries at the end are the useful diagnostic signal.
- `npm run prettier:check` currently reports unrelated formatting drift outside this issue. The files changed for this issue pass a direct `npx prettier --check`.
