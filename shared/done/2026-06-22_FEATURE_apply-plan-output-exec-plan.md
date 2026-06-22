# Add Text Apply Plan Output

This ExecPlan is a living document. Keep Progress, Decisions, Risks, and Lessons Learned up to date during execution.

## Purpose / Big Picture

Add `seed4j apply <module> --plan` so humans and agents can inspect which module parameter values Seed4J will use before applying a module, including incomplete plans. The command explains whether each resolved value came from explicit CLI input, project history, or module metadata defaults, and lists required parameters that still need CLI input or project history. This is a dry-run inspection feature: no files, commits, or history entries are created.

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

- Decision: Implement text-only `--plan`; exclude JSON and `--format`.
  Rationale: User explicitly scoped this phase to no JSON.
  Date/Author: 2026-06-22 / Codex

- Decision: Use the same required validation as normal apply.
  Rationale: Required values from history are accepted; missing means absent from both CLI and history.
  Date/Author: 2026-06-22 / Codex

- Decision: `--plan` renders incomplete plans instead of failing Picocli required validation.
  Rationale: Agents and humans need to see which values are still missing before applying the module; normal `apply` keeps the Picocli missing-options error and exit code 2. Mandatory metadata defaults are not treated as satisfying required apply input because normal `apply` does not inject them.
  Date/Author: 2026-06-22 / Codex

- Decision: Keep new planning model in `command/infrastructure/primary`.
  Rationale: It models CLI inspection and Picocli option naming, not core Seed4J business rules.
  Date/Author: 2026-06-22 / Codex

- Decision: Validate the feature through CLI behavior tests in `Seed4JCommandsFactoryTest`, not one test class per resolver/renderer.
  Rationale: The requested TDD skill requires tests to follow public behavior and stable contracts rather than production topology.
  Date/Author: 2026-06-22 / Codex

## Risks And Mitigations

- Risk: Accidentally changing normal apply semantics.
  Mitigation: Normal apply still validates against and passes the explicit-over-history merged module parameters.

- Risk: Treating mandatory metadata defaults as complete required input.
  Mitigation: `--plan` lists mandatory parameters as missing unless they come from explicit CLI input or project history; display defaults remain informational for non-mandatory values.

- Risk: Treating metadata defaults as runtime guarantees.
  Mitigation: Defaults are displayed for explainability only and are not injected into normal apply parameters.

- Risk: `--plan` accidentally writes files or commits.
  Mitigation: CLI tests assert no history action and no new commit for plan execution.

- Risk: Operational CLI options leak into module parameter maps.
  Mitigation: `--project-path`, `--commit`, and `--plan` are excluded from module parameter extraction.

## Validation Strategy

1. Driven with focused RED/GREEN CLI behavior tests.
2. `./mvnw -Dtest=Seed4JCommandsFactoryTest test` passed on 2026-06-22 for the original text plan output.
3. `./mvnw -Dtest=Seed4JCommandsFactoryTest test` passed on 2026-06-22 after adding incomplete plan output.
4. `npm run prettier:check` passed on 2026-06-22 after incomplete plan output changes.
5. `npm run prettier:check` passed on 2026-06-22 after installing pinned Node dependencies with `npm ci`.
6. Full `./mvnw clean verify` intentionally not run by the agent; ask the user to run it locally and report exit code plus relevant failures.

## Lessons Learned

- The first phase already added known completion candidates. The second phase must preserve value source before `ModuleParameters.merge(...)`, because the merged map cannot distinguish explicit input from project history.
- `GitTestUtil.getCommits(...)` returns commit log text, not a collection; tests comparing commit stability should compare the returned text directly.
- The `init` module definition order starts with `projectName`, then `baseName`; plan output follows module property definition order.
- The `init` module metadata exposes defaults for mandatory `projectName` and `baseName`, but normal `apply` still requires CLI or project history. Incomplete plan output must follow normal apply semantics and list those mandatory parameters as missing.
