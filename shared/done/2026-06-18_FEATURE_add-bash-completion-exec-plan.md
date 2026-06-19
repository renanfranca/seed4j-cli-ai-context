# Add Bash Completion Support

This ExecPlan is a living document. Keep `Progress`, `Decisions`, `Risks`, and `Lessons Learned` up to date as work advances.

## Purpose / Big Picture

Add `seed4j completion bash`, a user-visible command that prints a static Bash completion script for the currently active Seed4J runtime. The script helps humans and LLM-driven terminal agents discover command names, generated module slugs, and option names without invoking Java on every TAB press. Users can observe the behavior by running `seed4j completion bash` and installing the printed script into their Bash completion directory.

Safety boundary: This task is limited to authorized, defensive maintenance of this repository. Do not provide offensive guidance or policy-bypassing instructions.

## Scope

In scope: add the `completion bash` command, generate static Bash from the active picocli `CommandSpec` tree, include available commands, nested subcommands, generated `apply` module slugs, regular options, and negated option names such as `--no-commit`, document installation and regeneration guidance, and validate with targeted tests.

Out of scope: option value completion, Zsh or Fish completion, automatic shell installation, Seed4J core API changes, and dynamic completion that launches Java for each completion request.

## Definitions

`CommandSpec` is picocli's runtime model for a command, its subcommands, and its options.

`Static Bash completion script` means a generated Bash function containing command and option words at generation time. It does not call back into `seed4j` during tab completion.

`Available command` means a command or subcommand present in the assembled CLI command tree and therefore available to the completion script.

`Negated option` means picocli's generated negative alias for a boolean option, such as `--no-commit` for `--commit`.

## Existing Context

The CLI command tree is assembled in `src/main/java/com/seed4j/cli/command/infrastructure/primary/Seed4JCommandsFactory.java` and rooted by `Seed4JCommand`. Generated `apply` subcommands come from `ApplyModuleSubCommandsFactory` and are visible or hidden according to runtime resources and hidden-resource configuration. Existing tests in `Seed4JCommandsFactoryTest`, `ExternalConfigTest`, and pre-Spring bootstrap tests exercise the command factory path and runtime extension behavior.

## Desired End State

Running `seed4j completion bash` prints only a Bash script to stdout. The script completes top-level commands including `list`, `apply`, `extension`, and `completion`; nested subcommands including `completion bash`; visible module slugs under `apply`; command options including module options; and both regular and negated boolean option names. Hidden modules are absent. Extension runtime completion reflects extension-provided module slugs because the script is generated from the active runtime command tree.

## Milestones

### Milestone 1 - Completion Command Path

#### Goal

Expose `seed4j completion bash` through the real command factory and verify the public command path emits a Bash completion script.

#### Changes

- [ ] Edit `src/test/java/com/seed4j/cli/command/infrastructure/primary/Seed4JCommandsFactoryTest.java` to specify the new public command behavior.
- [ ] Add `CompletionCommand` and `BashCompletionCommand` under `src/main/java/com/seed4j/cli/command/infrastructure/primary`.
- [ ] Wire the new command through `Seed4JCommand` / `Seed4JCommandsFactory` using the existing command-tree pattern.
- [ ] Add a package-private script generator in the primary CLI package if needed by the command implementation.

#### Validation

- [ ] Command: `./mvnw -Dtest=Seed4JCommandsFactoryTest test`
- [ ] Expected result: the suite passes and the completion command output contains the expected command and option candidates.

#### Acceptance Criteria

- [ ] `seed4j completion bash` is reachable through the real CLI command factory.
- [ ] Output is Bash script content and does not include explanatory prose.

### Milestone 2 - Static Script Semantics

#### Goal

Ensure generated Bash handles nested commands, generated module slugs, regular options, negated options, and omits option value completion.

#### Changes

- [ ] Add focused behavior coverage for the generator through a stable package-level API or public command path.
- [ ] Implement traversal of available `CommandSpec` subcommands and options.
- [ ] Include `CommandSpec.negatedOptionsMap()` values in option candidates.
- [ ] Keep generated Bash portable with case statements and simple word lists.

#### Validation

- [ ] Command: `./mvnw -Dtest=Seed4JCommandsFactoryTest,BashCompletionScriptGeneratorTest test`
- [ ] Expected result: all completion semantics pass, including `--no-commit` and no option-value candidates.

#### Acceptance Criteria

- [ ] Top-level, nested, and module-specific candidates are present where expected.
- [ ] Option names are completed but option values are not.

### Milestone 3 - Runtime Visibility and Extension Coverage

#### Goal

Prove hidden modules stay absent and extension-mode completion includes extension-provided modules.

#### Changes

- [ ] Extend hidden-resource coverage in `ExternalConfigTest` or the most appropriate existing public-path suite.
- [ ] Extend pre-Spring/runtime coverage in `PreSpringBootstrapPrimaryTest` or existing extension runtime tests.
- [ ] Adjust production wiring only if the command is not included in all runtime assembly paths.

#### Validation

- [ ] Command: `./mvnw -Dtest=Seed4JCommandsFactoryTest,BashCompletionScriptGeneratorTest,ExternalConfigTest,PreSpringBootstrapPrimaryTest test`
- [ ] Expected result: targeted coverage passes for standard and extension runtime completion.

#### Acceptance Criteria

- [ ] Hidden module slugs do not appear in completion output.
- [ ] Extension-provided module slugs do appear when the extension runtime is active.

### Milestone 4 - Documentation and Formatting

#### Goal

Document install and regeneration guidance, then run repository formatting checks.

#### Changes

- [ ] Update `README.md` with the `completion bash` command and install example.
- [ ] Update `documentation/Commands.md` with static-runtime and regeneration guidance.
- [ ] Run Prettier check and update formatting if required.

#### Validation

- [ ] Command: `npm run prettier:check`
- [ ] Expected result: formatting check passes.

#### Acceptance Criteria

- [ ] Users know how to install the Bash completion script.
- [ ] Users know to regenerate after runtime extension or hidden-resource configuration changes.

## Progress

- [x] ExecPlan created
- [x] Milestone 1 started
- [x] Milestone 1 completed
- [x] Milestone 2 started
- [x] Milestone 2 completed
- [x] Milestone 3 started
- [x] Milestone 3 completed
- [x] Milestone 4 started
- [x] Milestone 4 completed
- [x] 2026-06-18 cleanup started to remove speculative `CommandSpec.hidden` filtering from Bash completion
- [x] 2026-06-18 `BashCompletionScriptGeneratorTest` updated to remove artificial hidden subcommand setup
- [x] 2026-06-18 `BashCompletionScriptGenerator` updated to traverse the assembled command tree without checking `usageMessage().hidden()`
- [x] 2026-06-18 cleanup validation completed with focused generator test, public hidden-config checkpoint, command factory checkpoint, and Prettier check

## Decisions

- Decision: Generate a static Bash script from picocli `CommandSpec` instead of using picocli's generated Bash script.
  Rationale: The feature requires explicit control over candidates, especially negated options such as `--no-commit`, and avoids launching Java during completion.
  Date/Author: 2026-06-18 / Codex

- Decision: Keep completion under primary CLI infrastructure.
  Rationale: Completion is presentation of the CLI command tree, not domain behavior, and should not introduce filesystem or runtime layout concerns into domain/application layers.
  Date/Author: 2026-06-18 / Codex

- Decision: Remove the speculative `CommandSpec.usageMessage().hidden()` filter from Bash completion traversal.
  Rationale: Hidden Seed4J modules are filtered earlier by `Seed4JModulesResources`; adding a generic picocli hidden-subcommand contract in the generator created behavior the CLI does not currently use and was only covered by an artificial unit-test setup.
  Date/Author: 2026-06-18 / Codex

## Risks and Mitigations

- Risk: Tests could mirror the generator implementation instead of user-visible behavior.
  Mitigation: Prefer command factory/public command tests and use generator tests only for the stable script-generation contract.

- Risk: Hidden module filtering may be bypassed if completion builds candidates from raw resources instead of generated command specs.
  Mitigation: Traverse the already assembled `CommandSpec` tree and rely on `Seed4JModulesResources` to keep hidden modules out of that tree.

- Risk: Bash escaping errors could produce invalid completion scripts for unusual module or option names.
  Mitigation: Quote shell words safely and keep script structure simple.

## Validation Strategy

1. Run targeted tests after each behavior cycle, starting with `Seed4JCommandsFactoryTest`.
2. Run `./mvnw -Dtest=Seed4JCommandsFactoryTest,BashCompletionScriptGeneratorTest,ExternalConfigTest,PreSpringBootstrapPrimaryTest test` after implementation.
3. Run `npm run prettier:check` after code and documentation edits.
4. Ask the user to run `./mvnw clean verify` locally as the final full gate.

Cleanup validation on 2026-06-18:

- `./mvnw -Dtest=BashCompletionScriptGeneratorTest test` passed after removing the artificial hidden subcommand scenario and again after production cleanup.
- `./mvnw -Dtest=ExternalConfigTest,Seed4JCommandsFactoryTest test` passed, confirming hidden modules remain absent through the public external configuration path.
- `npm run prettier:check` passed.

## Rollout and Recovery

This is an additive CLI command. If problems appear after release, revert the new completion command wiring, generator, tests, and documentation while leaving existing `list`, `apply`, and `extension` behavior unchanged. Users who installed a generated script can remove `~/.local/share/bash-completion/completions/seed4j` or regenerate after a fixed release.

## Lessons Learned

- The first public command-path test failed as expected with unmatched `completion bash`, then passed after adding command wiring and storing the generated command spec on `BashCompletionCommand` because `wrapWithoutInspection` does not inject ``.
- Generator behavior, hidden-resource behavior, extension-runtime behavior, documentation, Prettier, and the targeted Maven suite were completed on 2026-06-18.
- Review found that option names would have been suggested in option value positions; the generator now emits value-option cases and returns no candidates after value-taking options.
- The cleanup test stayed green before production changes because the artificial hidden subcommand assertion had been the only direct test for `usageMessage().hidden()`. Hidden module behavior remains covered through `ExternalConfigTest`, which exercises the public command factory path.
