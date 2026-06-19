# Refactor Bash Completion Generator Streams

This ExecPlan is a living document. Keep `Progress`, `Decisions`, `Risks`, and `Lessons Learned` up to date as work advances.

## Purpose / Big Picture

Improve the readability of the Bash completion script generator by replacing simple local mutable accumulation with stream and collector expressions where they make intent clearer. The generated Bash script must stay identical in behavior: command candidates, option candidates, value-taking options, negated options, ordering, and quoting continue to work through the existing CLI completion path.

## Scope

In scope: local refactoring of `src/main/java/com/seed4j/cli/command/infrastructure/primary/BashCompletionScriptGenerator.java`, focused validation of existing Bash completion behavior, and living updates to this ExecPlan.

Out of scope: changing CLI behavior, changing help text, changing command registration, adding new completion features, restructuring recursion, extracting new production components, or running `./mvnw clean verify` automatically.

## Definitions

Bash completion script: the generated shell code printed by `seed4j completion bash` and used by Bash to suggest `seed4j` subcommands and options.

Candidate path: the command path used in generated `case` statements, such as empty root path, `apply`, or `apply init`.

Value option: an option that expects a user-provided value, such as `--project-path`, and therefore should suppress candidate completion for the following word.

## Existing Context

`BashCompletionScriptGenerator` currently builds candidate maps recursively from picocli `CommandSpec` objects. Some methods already use streams, while others use local mutable collections: an `ArrayList` in `collectCandidates`, a `TreeMap` in `subcommandsByName`, and a `StringBuilder` in `caseStatements`. `BashCompletionScriptGeneratorTest` verifies the stable behavior of the generated script through the generator API, and `Seed4JCommandsFactoryTest#shouldPrintBashCompletionScript` verifies the public CLI path prints a Bash completion script.

## Desired End State

The generator keeps the same behavior while using streams/collectors for simple list, map, and string assembly. The recursion remains straightforward. Existing behavior tests pass before and after the refactor, and the public CLI completion test remains green.

## Milestones

### Milestone 1 - Validate Existing Behavior

#### Goal

Confirm the current Bash completion behavior is green before changing production code.

#### Changes

- [ ] No production changes.
- [ ] Create this ExecPlan under the shared seed4j CLI AI context directory.

#### Validation

- [ ] Command: `./mvnw -Dtest=BashCompletionScriptGeneratorTest test`
- [ ] Expected result: test exits 0 and confirms generated script behavior remains currently valid.

#### Acceptance Criteria

- [ ] Existing generator behavior test passes before refactoring.

### Milestone 2 - Refactor Local Mutable Assembly

#### Goal

Replace simple mutable local assembly with streams/collectors without changing generated output.

#### Changes

- [ ] Edit `src/main/java/com/seed4j/cli/command/infrastructure/primary/BashCompletionScriptGenerator.java`.
- [ ] Build combined command/option candidates without an intermediate mutable `ArrayList`.
- [ ] Collect subcommands into a `TreeMap` using stream collectors, preserving ordering.
- [ ] Build generated `case` statements via `Collectors.joining()` instead of `StringBuilder`.
- [ ] Adjust imports.

#### Validation

- [ ] Command: `./mvnw -Dtest=BashCompletionScriptGeneratorTest test`
- [ ] Expected result: test exits 0 and generated script assertions still pass.

#### Acceptance Criteria

- [ ] Bash completion generator tests pass after the refactor.
- [ ] Refactor does not alter public API or CLI behavior.

### Milestone 3 - Public CLI Checkpoint

#### Goal

Verify the refactor remains valid through the CLI-facing command factory path.

#### Changes

- [ ] No extra production changes unless validation exposes a behavior issue.

#### Validation

- [ ] Command: `./mvnw -Dtest=Seed4JCommandsFactoryTest#shouldPrintBashCompletionScript test`
- [ ] Expected result: test exits 0 and confirms `completion bash` still prints the expected script markers and candidates.

#### Acceptance Criteria

- [ ] Public CLI completion test passes.

## Progress

- [x] ExecPlan created
- [x] Milestone 1 started
- [x] Milestone 1 completed
- [x] Milestone 2 started
- [x] Milestone 2 completed
- [x] Milestone 3 started
- [x] Milestone 3 completed

## Decisions

- Decision: Keep the refactor local and behavior-neutral.
  Rationale: The user asked for more elegant stream usage in this file, and deeper recursion restructuring would increase risk without improving observable behavior.
  Date/Author: 2026-06-18 / Codex

- Decision: Do not add tests by default.
  Rationale: The existing tests already observe the generated script behavior and the public CLI path; adding tests for internal stream usage would mirror implementation details.
  Date/Author: 2026-06-18 / Codex

## Risks and Mitigations

- Risk: Stream refactoring could accidentally alter ordering.
  Mitigation: Preserve `TreeMap`, `sorted()`, and `distinct()` behavior, then rerun focused tests.

- Risk: Refactoring generated strings could alter whitespace or quoting.
  Mitigation: Keep the same `quote(...)` method and validate existing script substring assertions.

## Validation Strategy

1. Run `./mvnw -Dtest=BashCompletionScriptGeneratorTest test` before changes.
2. Refactor while the behavior test is green.
3. Run `./mvnw -Dtest=BashCompletionScriptGeneratorTest test` after changes.
4. Run `./mvnw -Dtest=Seed4JCommandsFactoryTest#shouldPrintBashCompletionScript test` as the public CLI checkpoint.
5. Do not run `./mvnw clean verify` automatically.

## Rollout and Recovery

This is an internal refactor with no migration or release toggle. If validation fails, inspect the generated script diff and revert only the refactor hunks that changed behavior while preserving unrelated user changes.

## Lessons Learned

- Existing behavior tests were sufficient for this behavior-neutral refactor; adding implementation-detail tests would have reduced refactor freedom.
- Baseline focused generator test passed before refactoring on 2026-06-18.
- Focused generator test passed after refactoring on 2026-06-18.
- Public CLI checkpoint passed on 2026-06-18.
- `npm run prettier:check` passed on 2026-06-18.
