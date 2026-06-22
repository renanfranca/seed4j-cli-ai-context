# Add Bash Completion for Option Values

This ExecPlan is a living document. Keep `Progress`, `Decisions`, `Risks`, and `Lessons Learned` up to date as work advances.

## Purpose / Big Picture

Deliver Bash completion for option values so users can press `<TAB>` after options like `--project-name` and receive known defaults from Seed4J module metadata. The generated Bash completion script includes value completion by default, while `--no-complete-values` generates a script without option value candidates. Safety boundary: This task is limited to authorized, defensive maintenance of this repository.

## Scope

In scope: Bash completion only, static candidates from CLI and module metadata, `--option <TAB>`, `--option=<TAB>`, tests, README, and command documentation.

Out of scope: filesystem path completion, history-derived values, heuristic enum parsing, zsh or fish completion, and changes to the upstream Seed4J core project.

## Definitions

Value completion means shell suggestions for the value expected by an option.

Completion candidate means a value attached to a picocli `OptionSpec` through `completionCandidates(...)`.

Value-disabled completion means completion that includes commands, subcommands, option names, and negated options, but not option values.

## Existing Context

`src/main/java/com/seed4j/cli/command/infrastructure/primary/BashCompletionScriptGenerator.java` collects command and option names and currently returns no suggestions when the previous token is an option requiring a value.

`src/main/java/com/seed4j/cli/command/infrastructure/primary/ApplyModuleSubCommand.java` builds module options from `Seed4JModulePropertyDefinition`. That upstream contract includes `defaultValue()`. Mandatory module properties must remain mandatory at execution time, so those defaults must not become `OptionSpec.defaultValue(...)`.

`src/main/java/com/seed4j/cli/command/infrastructure/primary/BashCompletionCommand.java` exposes `seed4j completion bash` and `seed4j completion bash --install`.

## Desired End State

`seed4j completion bash` and `seed4j completion bash --install` generate richer completion by default. `seed4j completion bash --no-complete-values` and install with the same flag generate completion without option value candidates. Known module defaults such as `Seed4J Sample Application`, `seed4jSampleApplication`, `npm`, and `.` are available as single completion candidates where applicable.

## Milestones

### Milestone 1 - RED tests for generator behavior

#### Goal

Specify value completion through the stable Bash completion generator behavior without testing private parsing details.

#### Changes

- [ ] Edit `src/test/java/com/seed4j/cli/command/infrastructure/primary/BashCompletionScriptGeneratorTest.java` to add failing behavior coverage for option value candidates, candidates with spaces, `--option value`, `--option=value`, and disabled value completion.

#### Validation

- [ ] Command: `./mvnw -Dtest=BashCompletionScriptGeneratorTest test`
- [ ] Expected result: tests fail for missing value completion.

#### Acceptance Criteria

- [ ] The missing behavior is demonstrated by failing tests.

### Milestone 2 - Generator implementation

#### Goal

Generate Bash script logic that completes explicit option candidates while preserving the disabled value-completion mode.

#### Changes

- [ ] Edit `src/main/java/com/seed4j/cli/command/infrastructure/primary/BashCompletionScriptGenerator.java` to support value completion enabled or disabled.
- [ ] Collect `OptionSpec.completionCandidates()` by command path and option name.
- [ ] Emit Bash logic that preserves values with spaces as one candidate.

#### Validation

- [ ] Command: `./mvnw -Dtest=BashCompletionScriptGeneratorTest test`
- [ ] Expected result: generator tests pass.

#### Acceptance Criteria

- [ ] Value candidates complete for separated and equals-assigned values.
- [ ] Completion without value candidates remains unchanged when value completion is disabled.

### Milestone 3 - CLI metadata and flag wiring

#### Goal

Expose value completion through the real `seed4j completion bash` command and attach candidates from module metadata without changing command execution requirements.

#### Changes

- [ ] Edit `src/main/java/com/seed4j/cli/command/infrastructure/primary/ApplyModuleSubCommand.java` to attach completion candidates from `Seed4JModulePropertyDefinition.defaultValue()` and `--project-path` default `.`.
- [ ] Edit `src/main/java/com/seed4j/cli/command/infrastructure/primary/BashCompletionCommand.java` to add negatable `--complete-values` defaulting to true and pass the selected mode to the generator.
- [ ] Update tests in `BashCompletionInstallationCommandsTest` and `Seed4JCommandsFactoryTest` as needed.

#### Validation

- [ ] Command: `./mvnw -Dtest=BashCompletionScriptGeneratorTest,BashCompletionInstallationCommandsTest,Seed4JCommandsFactoryTest test`
- [ ] Expected result: focused tests pass.

#### Acceptance Criteria

- [ ] `completion bash` contains value completion.
- [ ] `completion bash --no-complete-values` does not include value completion behavior.
- [ ] Missing mandatory apply options still fail.

### Milestone 4 - Documentation and formatting

#### Goal

Document default value completion, the value-completion opt-out flag, static regeneration, and scope limits.

#### Changes

- [ ] Edit `README.md`.
- [ ] Edit `documentation/Commands.md`.

#### Validation

- [ ] Command: `npm run prettier:check`
- [ ] Expected result: formatting check passes.

#### Acceptance Criteria

- [ ] Documentation no longer claims option values are unsupported.
- [ ] Documentation describes `--no-complete-values` and static generation limits.

## Progress

- [x] ExecPlan file created
- [x] Milestone 1 started
- [x] Milestone 1 completed
- [x] Milestone 2 started
- [x] Milestone 2 completed
- [x] Milestone 3 started
- [x] Milestone 3 completed
- [x] Milestone 4 started
- [x] Milestone 4 completed

## Decisions

- Decision: Enable value completion by default and provide `--no-complete-values`.
  Rationale: This is simplest for users while keeping an explicit opt-out for scripts without value candidates.
  Date/Author: 2026-06-19 / Codex

- Decision: Use `completionCandidates(...)`, not `defaultValue(...)`, for module defaults.
  Rationale: Mandatory module options must remain mandatory during command execution.
  Date/Author: 2026-06-19 / Codex

- Decision: Limit v1 to explicit metadata and defaults.
  Rationale: Avoid guesses and keep completion static, predictable, and LLM-friendly.
  Date/Author: 2026-06-19 / Codex

- Decision: Implement the opt-out as an explicit `--no-complete-values` option rather than relying on Picocli's generated negatable option value.
  Rationale: In this command setup, Picocli accepted the generated negated spelling but the positive option value still read as enabled while generating the script. An explicit opt-out keeps the user-facing command stable and testable.
  Date/Author: 2026-06-19 / Codex

## Risks and Mitigations

- Risk: Bash token splitting breaks values with spaces.
  Mitigation: Test `Seed4J Sample Application` as one candidate and emit candidates in a safe quoted or line-oriented form.

- Risk: Completion defaults accidentally become runtime defaults.
  Mitigation: Keep execution defaults unchanged and preserve missing-required-option tests.

- Risk: Some workflows may prefer no value suggestions.
  Mitigation: Provide `--no-complete-values` and document it.

## Validation Strategy

1. Run focused generator tests.
2. Run focused command, install, and factory tests.
3. Run `npm run prettier:check`.
4. Ask for the full local gate: `./mvnw clean verify`.

## Rollout and Recovery

Rollout is a normal CLI release. Recovery is to regenerate completion with `seed4j completion bash --no-complete-values --install` or revert the focused CLI completion changes.

## Lessons Learned

- Seed4JModulePropertyDefinition.defaultValue() already exists in the upstream Seed4J contract.
- Some real defaults contain spaces, so command and option candidate formatting cannot be reused unchanged for values.
- Bash `printf '%s\n'` with no arguments emits one blank line, so Bash-level completion tests must avoid printing when `COMPREPLY` is empty.
- Picocli's programmatic negatable option parsing needs care when reading option values from manually built command specs.
- Validation passed on 2026-06-19 with `./mvnw -Dtest=BashCompletionScriptGeneratorTest,BashCompletionInstallationCommandsTest,Seed4JCommandsFactoryTest test`.
- Validation passed on 2026-06-19 with `npm run prettier:check` after installing Node dependencies with `npm ci`.
- Wording cleanup passed on 2026-06-19 with `./mvnw -Dtest=Seed4JCommandsFactoryTest,BashCompletionInstallationCommandsTest test` and `npm run prettier:check`; `--no-complete-values` is now described neutrally as generation without option value candidates.
