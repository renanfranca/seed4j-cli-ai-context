# Install Bash Completion

This ExecPlan is a living document. Keep `Progress`, `Decisions`, `Risks`, and `Lessons Learned` up to date as work advances.

## Purpose / Big Picture

Add `seed4j completion bash --install`, a user-visible command that installs the generated Bash completion script at the standard per-user Bash completion location. Users can observe the behavior by running the command and seeing the installed file path plus the exact `source ~/.local/share/bash-completion/completions/seed4j` instruction for the current shell. The existing `seed4j completion bash` output remains unchanged so scripts can still redirect or inspect the generated completion script manually.

Safety boundary: This task is limited to authorized, defensive maintenance of this repository. Do not provide offensive guidance or policy-bypassing instructions.

## Scope

In scope: add the `--install` option to `completion bash`, write the generated static script to `~/.local/share/bash-completion/completions/seed4j`, create the parent directory when needed, overwrite an existing generated script, translate filesystem failures into a command-domain exception, document the command, and validate with focused tests.

Out of scope: Zsh or Fish completion, custom install paths, privileged system-wide installation, shell auto-reload, changing generated Bash script semantics, and running the repository full `./mvnw clean verify` gate automatically.

## Definitions

`BashCompletionScript` is a command-domain value object containing the generated Bash script text.

`BashCompletionInstallationPath` is a command-domain value object for the user-visible installed script path.

`BashCompletionInstaller` is a command-domain port named after the business capability of installing Bash completion. Its filesystem implementation lives in secondary infrastructure.

`Standard per-user Bash completion location` means `~/.local/share/bash-completion/completions/seed4j`, derived from `user.home`.

## Existing Context

The command tree is assembled in `src/main/java/com/seed4j/cli/command/infrastructure/primary/Seed4JCommandsFactory.java`. `src/main/java/com/seed4j/cli/command/infrastructure/primary/BashCompletionCommand.java` currently prints `new BashCompletionScriptGenerator().generate(commandSpec.root())` to stdout. `BashCompletionScriptGenerator` is intentionally in primary infrastructure because it presents picocli `CommandSpec` state. The command context already has `domain`, `application`, and `infrastructure.secondary` packages used by runtime extension use cases.

## Desired End State

`seed4j completion bash` still prints only the generated Bash completion script. `seed4j completion bash --install` writes that exact generated script to `~/.local/share/bash-completion/completions/seed4j`, creates the completion directory when missing, overwrites stale content, returns exit code `0`, and prints concise instructions including the installed path and `source ~/.local/share/bash-completion/completions/seed4j`. Filesystem failures are translated to `BashCompletionInstallationException` with a useful message.

## Milestones

### Milestone 1 - Installer Contract and Filesystem Behavior

#### Goal

Specify and implement the stable installer behavior independently from picocli command parsing.

#### Changes

- [x] Add `BashCompletionScript`, `BashCompletionInstallationPath`, `BashCompletionInstallationResult`, `BashCompletionInstaller`, and `BashCompletionInstallationException` under `src/main/java/com/seed4j/cli/command/domain`.
- [x] Add `BashCompletionInstallApplicationService` under `src/main/java/com/seed4j/cli/command/application`.
- [x] Add `FileSystemBashCompletionInstaller` under `src/main/java/com/seed4j/cli/command/infrastructure/secondary`.
- [x] Add focused behavior tests for directory creation, overwrite behavior, result path, and failure translation.

#### Validation

- [x] Command: `./mvnw -Dtest=FileSystemBashCompletionInstallerTest test`
- [x] Expected result: focused installer tests pass.

#### Acceptance Criteria

- [x] Missing install directories are created.
- [x] Existing completion file content is overwritten.
- [x] The returned result exposes the installed path.
- [x] I/O failures are translated to `BashCompletionInstallationException`.

### Milestone 2 - CLI Install Option

#### Goal

Expose installation through `seed4j completion bash --install` while preserving raw script printing without `--install`.

#### Changes

- [x] Update `BashCompletionCommand` to accept `--install`.
- [x] Generate the script from the current root `CommandSpec` and pass it to `BashCompletionInstallApplicationService`.
- [x] Print exact shell reload guidance after successful install.
- [x] Update command fixture and command factory tests for the new dependency.

#### Validation

- [x] Command: `./mvnw -Dtest=Seed4JCommandsFactoryTest,BashCompletionInstallationCommandsTest test`
- [x] Expected result: command behavior tests pass.

#### Acceptance Criteria

- [x] `completion bash` script output stays unchanged.
- [x] `completion bash --install` installs and prints `source ~/.local/share/bash-completion/completions/seed4j`.
- [x] The command exits successfully through the public command path.

### Milestone 3 - Documentation and Focused Gate

#### Goal

Document the install command and run focused validation for the affected surface.

#### Changes

- [x] Update `README.md`.
- [x] Update `documentation/Commands.md`.
- [x] Run focused Maven tests and Prettier check.

#### Validation

- [x] Command: `./mvnw -Dtest=BashCompletionScriptGeneratorTest,FileSystemBashCompletionInstallerTest,BashCompletionInstallationCommandsTest,Seed4JCommandsFactoryTest,ExternalConfigTest,HexagonalArchTest test`
- [x] Command: `npm run prettier:check`
- [x] Expected result: all focused checks pass.

#### Acceptance Criteria

- [x] Documentation describes `--install` as the normal install path.
- [x] Documentation still explains manual redirection and regeneration.
- [x] Focused validation passes.

## Progress

- [x] ExecPlan created
- [x] Milestone 1 started
- [x] Milestone 1 completed
- [x] Milestone 2 started
- [x] Milestone 2 completed
- [x] Milestone 3 started
- [x] Milestone 3 completed

## Decisions

- Decision: Keep generation from picocli `CommandSpec` in primary infrastructure and move only installation behind an application service and domain port.
  Rationale: Script generation is presentation of the CLI command tree, while writing to Bash's user completion directory is an external filesystem capability.
  Date/Author: 2026-06-18 / Codex

- Decision: Do not use `Seed4JCliHome` for this command-context feature.
  Rationale: The standard Bash completion directory is a shell integration location, not the Seed4J CLI home concept. `user.home` will be read at infrastructure composition/configuration boundaries and kept out of application/domain.
  Date/Author: 2026-06-18 / Codex

- Decision: Give the filesystem installer a `Path userHome` constructor and create the production bean from Spring configuration.
  Rationale: This keeps `user.home` reading at the infrastructure wiring boundary while allowing deterministic filesystem behavior tests without mutating global JVM state.
  Date/Author: 2026-06-18 / Codex

## Risks and Mitigations

- Risk: The install path could leak filesystem layout into domain/application as hidden operational state.
  Mitigation: Keep the concrete `~/.local/share/bash-completion/completions/seed4j` resolution inside secondary infrastructure and expose only the installed user-visible path in the result.

- Risk: Tests could couple to implementation topology instead of user behavior.
  Mitigation: Use installer tests for the stable domain port behavior and command tests through the picocli public path.

- Risk: A child process cannot reload the parent shell after installing completion.
  Mitigation: Print the exact `source ~/.local/share/bash-completion/completions/seed4j` instruction.

## Validation Strategy

1. Add one behavior test at a time and run the focused relevant Maven test suite.
2. Run a public CLI command-path checkpoint after wiring `--install`.
3. Run `./mvnw -Dtest=BashCompletionScriptGeneratorTest,FileSystemBashCompletionInstallerTest,BashCompletionInstallationCommandsTest,Seed4JCommandsFactoryTest,ExternalConfigTest,HexagonalArchTest test`.
4. Run `npm run prettier:check`.
5. Ask the user to run `./mvnw clean verify` locally before merge.

Validation completed on 2026-06-18:

- `./mvnw -Dtest=FileSystemBashCompletionInstallerTest test` passed with 2 tests.
- `./mvnw -Dtest=Seed4JCommandsFactoryTest,BashCompletionInstallationCommandsTest test` passed with 27 tests.
- `./mvnw -Dtest=BashCompletionScriptGeneratorTest,FileSystemBashCompletionInstallerTest,BashCompletionInstallationCommandsTest,Seed4JCommandsFactoryTest,ExternalConfigTest,HexagonalArchTest test` passed with 51 tests.
- `npm run prettier:check` passed.

## Rollout and Recovery

This is additive. If a defect appears, revert the new `--install` option, installer types, secondary filesystem adapter, tests, and documentation. Users can remove an installed script with `rm ~/.local/share/bash-completion/completions/seed4j` or regenerate it with a fixed CLI.

## Lessons Learned

- The filesystem failure path can be tested deterministically by placing a regular file at `.local/share/bash-completion`, causing completion directory creation to fail without changing global permissions.
- `CommandSpec.wrapWithoutInspection` does not read picocli annotations, so `--install` must be registered with `OptionSpec` like other command-layer options in this codebase.
