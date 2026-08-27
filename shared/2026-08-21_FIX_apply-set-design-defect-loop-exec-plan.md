# Redesign Apply-Set Planning with Typed Facts and Policies

This ExecPlan is a living document. Keep `Progress`, `Decisions`, `Risks`, and `Lessons Learned` up to date as work advances.

## Purpose / Big Picture

Make the read-only `apply-set` planner express validation failures and parameter values with domain types instead of generic strings and `Object`. Existing command syntax and rendered output remain stable except for one new actionable diagnostic when a relevant project-history parameter has the wrong type. Deliver the redesign as independently green commits so each contract change and policy extraction can be reviewed or reverted in isolation, then close the remaining JaCoCo gaps through public CLI and catalog behavior while removing one proven unreachable internal guard.

## Scope

In scope are the `apply-set` planning domain model, application service, primary CLI adapter and renderer, secondary catalog and project-history adapters, the existing application-service and command behavior suites, and the two canonical documents describing command behavior and architectural boundaries.

Milestone 1 structures request-validation, property-conflict, and unused-explicit-parameter failures while deriving missing dependency and required-parameter invalidity from their detailed results. Milestone 2 introduces closed typed parameter values at every boundary and rejects relevant incompatible history. Milestone 3 separates property-definition reconciliation from parameter-value resolution. Milestone 5 covers the remaining observable type-boundary behavior and removes the unreachable requiring-module position check without changing valid planning behavior.

Out of scope are the single-module `apply` command, effective module-set application, completion changes, help changes, new property types, changes to Seed4J core, and changes to current description/default conflict wording or ordering.

Safety boundary: This task is limited to authorized, defensive maintenance of this repository. Do not provide offensive guidance or policy-bypassing instructions.

## Definitions

A planning fact is a structured domain value describing why a request is invalid without containing CLI labels, option spelling, or complete English diagnostics. A parameter value is one of the closed `STRING`, `INTEGER`, or `BOOLEAN` variants supported by Seed4J. A catalog literal is the original textual spelling of a Seed4J property default; it remains authoritative for conflict comparison, help, and completion even when its corresponding planning value is typed. Relevant history is a stored parameter whose key is required by at least one selected module and is not overridden explicitly. A public-path checkpoint executes the packaged CLI JAR against a temporary project directory and verifies exit code, output, and the read-only filesystem guarantee.

## Existing Context

The preceding review loop already isolated Picocli invocation state and extracted dependency and parameter planners. `ModuleSetPlanningProblem` is still a type-plus-list-of-strings tuple. Some strings contain `--kebab-case` CLI syntax or complete English conflict fragments, while missing dependency and missing required parameter failures are duplicated both as detailed plan results and generic problems.

`ExplicitModuleSetParameters`, `ModuleSetPlanningHistory`, and `ResolvedModuleSetParameter` expose raw `Object` values. `Seed4JModuleSetCatalog` transports every textual default unchanged, so an `INTEGER` default remains a `String`. `ProjectsModuleSetPlanningHistoryReader` transports arbitrary Java/JSON objects directly into the domain. The parameter planner accepts either representation without verifying that a relevant stored value matches the selected property definition.

The focused behavior suite is `ModuleSetPlanningApplicationServiceTest` plus the public CLI observations in `Seed4JCommandsFactoryTest`. It was green before this redesign on 2026-08-22.

## Desired End State

`ModuleSetPlanningProblem` is a sealed hierarchy of typed facts and `ModuleSetPlanningProblemType` no longer exists. Slugs and property keys remain domain types until the primary renderer converts them to presentation text. Plan validity derives missing dependencies and missing required parameters from detailed results rather than duplicated problems. `EXPLICIT_INPUT` names the domain source while the renderer still prints `explicit CLI input`.

All recognized explicit, history, default, and resolved parameter values use a sealed `ModuleSetParameterValue` with string, integer, and boolean variants. Catalog defaults retain both the typed value and exact original literal. The CLI, catalog, and history adapters perform conversions. Unsupported history values cross into the domain only as structured unsupported-type facts, never as `Object`. A relevant history type mismatch invalidates the plan, is not treated as missing, does not fall back to a default, and is rendered alphabetically by key with an override instruction. A correct explicit value wins over incompatible stored history, and irrelevant stored keys remain ignored.

Finally, one pure policy reconciles shared property definitions into immutable definitions and typed conflicts, while a separate value-resolution policy consumes only selected definitions, explicit parameters, and history parameter facts. Internal outcomes distinguish resolved, required missing, incompatible history, and optional without value. Public CLI coverage reaches complex, `null`, and boolean historical values, while the catalog contract proves strict boolean-default conversion. Dependency planning retains the missing-candidate guard but no longer checks for a requiring module that must already belong to the indexed execution order.

## Milestones

### Milestone 1 - Structure planning failures

#### Goal

Replace generic problem tuples and presentation strings with typed facts while preserving every current CLI diagnostic, conflict ordering, exit code, and planning result.

#### Changes

- [x] Replace `ModuleSetPlanningProblem` with a sealed hierarchy for duplicate modules, unknown modules, property conflicts, and unused explicit parameters; remove `ModuleSetPlanningProblemType`.
- [x] Carry `ModuleSetSlug` and `ModuleSetPropertyKey` in the facts and convert property keys to `--kebab-case` only in `ApplyModuleSetPlanRenderer`.
- [x] Model default and description conflicts as structured facts while preserving alphabetical value ordering and default-before-description rendering.
- [x] Rename `ModuleSetPropertySource.EXPLICIT_CLI` to `EXPLICIT_INPUT`, preserving renderer text `explicit CLI input`.
- [x] Remove missing dependency and missing required parameter entries from `problems`; derive `ModuleSetPlan.valid()` from detailed dependency and required-parameter results.
- [x] Keep documentation, completion, and help unchanged because this milestone changes internal representation only.

#### Validation

- [x] Command: `./mvnw -q -Dtest=ModuleSetPlanningApplicationServiceTest,Seed4JCommandsFactoryTest test`
- [x] Expected result: all focused application and CLI behavior tests pass without text changes.
- [x] Command: execute the packaged JAR public-path scenario described under Validation Strategy.
- [x] Expected result: exit code `0`, `Status: VALID`, `No changes were applied.`, and an untouched project directory.

#### Acceptance Criteria

- [x] No domain planning fact contains CLI option syntax or a complete English diagnostic.
- [x] Missing dependencies and required parameters invalidate plans without duplicated generic problems.
- [x] Commit `refactor(command): structure module set planning failures` contains only this milestone and an explanatory English body.

### Milestone 2 - Enforce parameter types

#### Goal

Represent the three supported parameter types explicitly and reject relevant incompatible project history with a clear CLI diagnostic while preserving precedence and catalog literal behavior.

#### Changes

- [x] Add sealed `ModuleSetParameterValue` variants for string, integer, and boolean values.
- [x] Change explicit parameters, recognized history parameters, defaults, and resolved parameters to use the typed value.
- [x] Make `ModuleSetPropertyDefaultValue` retain the typed value and original catalog literal; continue comparing and rendering literals so `"2"` and `"02"` remain different defaults.
- [x] Convert parsed Picocli values, catalog type-plus-default text, and history Java/JSON scalar values in their respective adapters.
- [x] Represent unsupported history types as structured facts without `Object` in the domain.
- [x] Add behavior-first tests, one cycle at a time, for typed integer defaults; relevant string-versus-integer history failure; explicit integer override; irrelevant incompatible history; relevant unsupported history; and aggregation ordered alphabetically.
- [x] Update `documentation/Commands.md` with the incompatibility rule and `documentation/hexagonal-architecture.md` with boundary conversions.

#### Validation

- [x] Command after every behavior cycle: `./mvnw -q -Dtest=ModuleSetPlanningApplicationServiceTest,Seed4JCommandsFactoryTest test`
- [x] Expected result: each new test fails for its predicted missing behavior and the complete focused suite becomes green after the minimum implementation.
- [x] Expected diagnostic: `Project history parameter type mismatch: indentSize expects INTEGER but history contains STRING; pass --indent-size to override the stored value`.
- [x] Command: repeat the packaged JAR public-path checkpoint.
- [x] Expected result: unchanged valid output and read-only behavior.

#### Acceptance Criteria

- [x] No planning-domain parameter map or resolved parameter exposes `Object`.
- [x] A relevant mismatch is invalid, not missing, and never resolved from a default.
- [x] A correctly typed explicit value overrides incompatible history, and irrelevant incompatible history is ignored.
- [x] Commit `fix(command): enforce module set parameter types` contains only this milestone and an explanatory English body.

### Milestone 3 - Separate parameter policies

#### Goal

Separate shared-definition reconciliation from value precedence and give each pure policy only the module-set facts it consumes.

#### Changes

- [x] Extract a pure shared-property-definition reconciliation policy returning immutable definitions and conflicts.
- [x] Change parameter planning to receive ordered selected modules, explicit parameters, and history parameter facts instead of the current execution-order/map/request/history bundle.
- [x] Represent each internal resolution as resolved, required missing, history incompatible, or optional without value.
- [x] Preserve property ordering, value precedence, mandatory-default informational behavior, conflict details, and rendered output.
- [x] Reuse service and public command behavior suites; do not add tests for extracted internal classes.
- [x] Keep canonical documents unchanged because milestone 2 already documents final behavior and boundary locations.

#### Validation

- [x] Command: `./mvnw -q -Dtest=ModuleSetPlanningApplicationServiceTest,Seed4JCommandsFactoryTest test`
- [x] Expected result: all focused tests pass with unchanged output.
- [x] Command: repeat the packaged JAR public-path checkpoint.
- [x] Expected result: exit code `0`, valid status, no changes message, and untouched directory.

#### Acceptance Criteria

- [x] Reconciliation and resolution are separate pure policies with immutable outputs.
- [x] The dense four-input parameter planner method no longer exists.
- [x] Commit `refactor(command): separate module set parameter policies` contains only this milestone and an explanatory English body.

### Milestone 4 - Post-green design consolidation and final validation

#### Goal

Review only the changed contracts and adjacent code for demonstrated structural risks, preserve behavior, and complete repository validation.

#### Changes

- [x] Apply `refactor-design` after requested behavior, focused suite, and public checkpoint are green.
- [x] Classify every materially reported candidate as defect, design risk, maintainability opportunity, or no action.
- [x] For a proven additional finding, use behavior TDD when needed, validate, and create its own explanatory commit; leave hypothetical functional changes as no action.
- [x] Reconcile this ExecPlan and canonical documentation with final behavior.

#### Validation

- [x] Command: `./mvnw test`
- [x] Expected result: 529 JUnit tests pass.
- [x] Command: `npm run prettier:check`
- [x] Expected result: all supported files are formatted.
- [x] Command: `habit-hooks --branch`
- [x] Expected result: every production finding introduced or aggravated by this redesign is analyzed and every demonstrated one is resolved; the command still exits `1` for the wider branch baseline and behavior-test size heuristics recorded below.
- [x] Command: repeat the packaged JAR public-path checkpoint after any post-green refactor.
- [x] Expected result: valid status and unchanged temporary project.

#### Acceptance Criteria

- [x] No unresolved actionable in-scope design finding remains.
- [x] The worktree contains no uncommitted tracked task changes.
- [x] The final handoff asks the user to run `./mvnw clean verify` locally and report its exit code.

### Milestone 5 - Close type-boundary coverage and remove the unreachable position guard

#### Goal

Cover the remaining real `apply-set` type conversions through existing public boundaries, then simplify dependency-position comparison according to the execution-order invariant without changing public contracts or valid command behavior.

#### Changes

- [x] Parameterize the existing public CLI test for unsupported project-history values so both a complex collection and `null` reach the structured mismatch diagnostic.
- [x] Add a public CLI scenario where a historical `BOOLEAN` value is incompatible with an `INTEGER` property and is rendered as an actionable invalid plan.
- [x] Add `Seed4JModuleSetCatalogTest` around `modules()` using a mocked Seed4J module service only as catalog input; prove `"true"` and `"false"` conversion and rejection of invalid `"yes"` with a clear diagnostic.
- [x] Remove only `requiredPosition != null` from `ModuleSetExecutionPositions.precedesAll`; retain the missing-candidate guard and add no artificial internal-state test.
- [x] Keep documentation unchanged because the public behavior and architecture rules are already canonical and this milestone only completes coverage plus an internal simplification.

#### Validation

- [x] Command: `./mvnw -q -Dtest=Seed4JCommandsFactoryTest,Seed4JModuleSetCatalogTest test`
- [x] Expected result: public CLI and catalog type-boundary behavior is green.
- [x] Command: `./mvnw -q -Dtest=ModuleSetPlanningApplicationServiceTest test`.
- [x] Expected result: all dependency planning scenarios remain green after removing the unreachable guard.
- [x] Command: `./mvnw test`
- [x] Expected result: all 534 JUnit tests pass.
- [x] Command: inspect `target/site/jacoco` for `ModuleSetDependencyPlanner`, `ModuleSetHistoryParameterValueType`, `ProjectsModuleSetPlanningHistoryReader`, and `Seed4JModuleSetCatalog`.
- [x] Expected result: no uncovered lines or branches remain in those four classes.
- [x] Command: `npm run prettier:check` and `habit-hooks --branch`
- [x] Expected result: formatting passes and every enforced in-scope Habit Hooks finding is resolved or explicitly analyzed; branch-wide Habit Hooks still exits `1` for the recorded baseline and inline behavior-test size heuristics.

#### Acceptance Criteria

- [x] Commit `test(command): cover module set type boundaries` contains only the coverage tests and an explanatory English body.
- [x] Commit `refactor(command): remove unreachable dependency position guard` contains only the guard removal and an explanatory English body documenting the invariant.
- [x] No private helper, Spring wiring, collaborator order, or impossible internal state is tested.
- [x] Post-green `refactor-design` review finds no unresolved actionable in-scope issue.

## Progress

- [x] Previous review-loop commits through `b76d434 refactor(command): extract module set parameter planner` are complete.
- [x] Authorization received to evolve internal public Java planning contracts while preserving CLI behavior.
- [x] Existing ExecPlan expanded with the three typed-redesign milestones and final review gate.
- [x] Baseline focused suite passed on 2026-08-22.
- [x] Milestone 1 started.
- [x] Milestone 1 implementation completed and focused suite passed on 2026-08-22.
- [x] Milestone 1 committed as `00ab7b3 refactor(command): structure module set planning failures`.
- [x] Milestone 2 started.
- [x] Milestone 2 behavior complete, documentation reconciled, and focused suite passed on 2026-08-22.
- [x] Milestone 2 committed as `786b407 fix(command): enforce module set parameter types`.
- [x] Milestone 3 started.
- [x] Milestone 3 behavior complete and focused suite passed on 2026-08-22.
- [x] Milestone 3 committed as `8600ca9 refactor(command): separate module set parameter policies`.
- [x] Packaged-JAR checkpoint green after all three planned commits: exit code `0`, `Status: VALID`, `No changes were applied.`, and zero project entries.
- [x] Initial post-green rubric review completed; Habit Hooks then supplied concrete evidence for additional maintainability findings.
- [x] Habit Hooks identified an actionable oversized resolution-aggregation flow; immutable summary and named precedence stages are green in the focused suite.
- [x] Post-green resolution-summary refactor committed as `c87ba84 refactor(command): summarize module set parameter resolutions`.
- [x] Habit Hooks identified mixed history-reader responsibilities; extraction of applied modules and exhaustive local parameter facts is green.
- [x] Post-green history-mapping refactor committed as `51d97ef refactor(command): isolate module set history mapping`.
- [x] Habit Hooks identified mixed plan/problem rendering responsibilities; the extracted primary problem renderer is green.
- [x] Post-green problem-renderer refactor committed as `9d19f06 refactor(command): extract module set problem renderer`.
- [x] Habit Hooks identified mixed top-level planning responsibilities; immutable request validation and selected-module planning are green.
- [x] Post-green top-level planner refactor committed as `435a93c refactor(command): separate module set request planning`.
- [x] Final validation commands complete; Maven, Prettier, and JAR checkpoint are green, while branch-wide Habit Hooks reports the documented baseline at exit `1`.
- [x] Milestone 5 coverage-closure request accepted and worktree confirmed clean on 2026-08-22.
- [x] Milestone 5 public type-boundary coverage started and focused suite passed after formatting.
- [x] Milestone 5 test commit created as `0773eb9 test(command): cover module set type boundaries`.
- [x] Milestone 5 unreachable guard removed and committed as `ef4cfe6 refactor(command): remove unreachable dependency position guard`.
- [x] Milestone 5 post-green review and final validation completed.

## Decisions

- Decision: Keep the ignored ExecPlan out of every commit while updating it continuously.
  Rationale: The user explicitly identified it as temporary context, and Git already ignores the path.
  Date/Author: 2026-08-22 / Codex
- Decision: Use three isolated commits with the exact requested subjects and English bodies adjusted only to match the final diff.
  Rationale: Each milestone changes a distinct concern and must remain independently green and revertible.
  Date/Author: 2026-08-22 / Codex
- Decision: Preserve existing dependency and missing-required detail records as canonical invalidity facts instead of wrapping them in another planning problem.
  Rationale: Those records already contain the structured facts the renderer needs; duplicating them risks inconsistent validity and diagnostics.
  Date/Author: 2026-08-22 / Codex
- Decision: Reserve catalog literal spelling separately from its typed planning value.
  Rationale: Numeric parsing must not erase meaningful textual distinctions used by conflict detection, help, and completion.
  Date/Author: 2026-08-22 / Codex
- Decision: Observe new type-mismatch behavior primarily through the public command suite and use the application-service suite for domain precedence and aggregation details.
  Rationale: Exit code and actionable wording belong to the CLI contract, while resolution semantics are faster and clearer at the application boundary.
  Date/Author: 2026-08-22 / Codex
- Decision: Keep property conflicts as one planning problem containing a sealed list of typed default or description conflicts.
  Rationale: The aggregate preserves the existing single `Property conflicts:` diagnostic line and ordering while each underlying fact retains domain keys and value objects.
  Date/Author: 2026-08-22 / Codex
- Decision: Model recognized history values separately from a list of key-only unsupported-history facts.
  Rationale: The three recognized scalar variants retain their values, while unsupported representations need only identify the affected key to drive mismatch behavior; neither contract exposes the external raw `Object`.
  Date/Author: 2026-08-22 / Codex
- Decision: Render unsupported history as `an unsupported value type` instead of exposing Java or JSON representation names.
  Rationale: The actionable contract is that the stored value cannot satisfy `STRING`, `INTEGER`, or `BOOLEAN`; adapter-specific class names would leak infrastructure and would not change the safe override action.
  Date/Author: 2026-08-22 / Codex
- Decision: Group recognized and unsupported history parameter facts in `ModuleSetHistoryParameters` before passing them to value resolution.
  Rationale: Dependency planning needs only applied module slugs, while value resolution needs only parameter facts; the nested type makes both policy inputs explicit without adding test-oriented overloads.
  Date/Author: 2026-08-22 / Codex
- Decision: Take no post-green action on the superficially duplicated global-property aggregation in `availableProperties()`.
  Rationale: Global Picocli metadata and selected-set reconciliation have intentionally different conflict semantics; reusing the selected-set policy would change help/completion behavior, which is explicitly outside scope.
  Date/Author: 2026-08-22 / Codex
- Decision: Take no post-green action on the theoretical overlap between recognized and unsupported history facts for the same key.
  Rationale: The exhaustive secondary-adapter switch emits exactly one fact per external history entry, no observable invalid runtime state exists, and another contract change would lack behavior evidence.
  Date/Author: 2026-08-22 / Codex
- Decision: Treat the Habit Hooks findings in `ModuleSetParameterPlanner.plan` and the value-precedence method as one actionable maintainability opportunity.
  Rationale: The planner still classified outcomes through three mutable accumulators, while the resolver expressed explicit input, recognized history, and absent history in one long branch. An immutable outcome summary and named precedence stages make those existing concepts explicit without changing behavior.
  Date/Author: 2026-08-22 / Codex
- Decision: Treat the oversized history-reader method as an actionable secondary-boundary finding.
  Rationale: It combined external lookup, applied-module extraction, scalar classification, and domain-history assembly. Named extractions plus a sealed local fact keep raw `Object` in one exhaustive mapping while leaving the domain contract unchanged.
  Date/Author: 2026-08-22 / Codex
- Decision: Treat the oversized plan renderer as an actionable primary-boundary finding.
  Rationale: Overall plan-section composition and exhaustive translation of typed planning problems have distinct inputs and reasons to change. A dedicated problem renderer keeps every English diagnostic and CLI spelling at the primary boundary while restoring one responsibility per renderer.
  Date/Author: 2026-08-22 / Codex
- Decision: Treat the oversized `ModuleSetPlanner.plan` method as an actionable orchestration finding.
  Rationale: Request validation and selected-module planning produce distinct immutable results and have separate collaborators. Naming both results removes mutable partial plan state while preserving problem and execution ordering.
  Date/Author: 2026-08-22 / Codex
- Decision: Take no action on remaining Habit Hooks method/file-size findings in `ModuleSetPlanningApplicationServiceTest` and `Seed4JCommandsFactoryTest`.
  Rationale: Each reported test method is one observable Given/When/Then scenario whose setup and assertions must remain explicit under repository testing rules. Splitting the broad historical test files is a separate documentation/topology task and would not validate this redesign more reliably.
  Date/Author: 2026-08-22 / Codex
- Decision: Report, rather than modify, the remaining branch-wide enforced findings outside the seven delivered commits.
  Rationale: They concern bootstrap launchers/tests, command-factory construction, dependency-planner code unchanged by this redesign, architecture/completion files, and other pre-existing branch work. Expanding into those areas would exceed authorization and mix unrelated changes into this delivery.
  Date/Author: 2026-08-22 / Codex
- Decision: Extend this existing redesign ExecPlan with a fifth milestone instead of creating a separate plan.
  Rationale: The requested coverage gaps and unreachable guard are the final validation residue of the same typed `apply-set` redesign and depend on its exact invariants.
  Date/Author: 2026-08-22 / Codex
- Decision: Treat `Seed4JModuleSetCatalog.modules()` as an intentional public adapter contract for boolean-default conversion.
  Rationale: The conversion is observable by callers of the domain port implementation, while private conversion helpers and Spring wiring are not stable behavior contracts.
  Date/Author: 2026-08-22 / Codex
- Decision: Classify the milestone 5 post-green review as `No action` for all inspected dimensions.
  Rationale: History and catalog conversions remain exhaustive at secondary boundaries, the new tests observe stable public contracts without test-only production seams, and requiring-module positions are guaranteed by the private execution-order construction path while the distinct missing-candidate case remains guarded.
  Date/Author: 2026-08-22 / Codex

## Risks and Mitigations

- Risk: Sealed problem variants could accidentally reorder or alter existing diagnostics.
  Mitigation: Preserve encounter order in the plan, encode default and description conflicts separately in their current order, and run the complete focused suite after the milestone.
- Risk: Typed defaults could normalize catalog literals and hide conflicts such as `"2"` versus `"02"`.
  Mitigation: Store and compare the original literal independently from the parsed value and extend the existing conflict behavior test.
- Risk: History can contain arbitrary values even though the planning domain must not expose `Object`.
  Mitigation: Convert recognized scalar variants in the secondary adapter and transport only a structured unsupported-type fact for every other representation.
- Risk: Incompatible history could be incorrectly treated as missing or defaulted.
  Mitigation: Model incompatibility as its own resolution outcome and test precedence through public behavior.
- Risk: Policy extraction could mirror production topology in tests.
  Mitigation: Reuse application-service and command tests; do not create test classes for extracted policies.
- Risk: Habit Hooks may report repository-wide baseline findings outside this feature.
  Mitigation: Analyze in-scope findings, report unrelated baselines without changing them, and never snooze findings without explicit authorization.
- Risk: Coverage work could drift into tests that mirror private conversion helpers or impossible dependency state.
  Mitigation: Observe history through command output and catalog defaults through `modules()`; remove the unreachable guard without fabricating an internal-state test.
- Risk: `null` cannot be represented by `Map.of` in the project-history fixture.
  Mitigation: Use a nullable map implementation at the existing Seed4J history boundary and keep assertions on the command result and rendered diagnostic.

## Validation Strategy

1. Run `./mvnw -q -Dtest=ModuleSetPlanningApplicationServiceTest,Seed4JCommandsFactoryTest test` before and after milestone 1 and milestone 3, and during every RED/GREEN behavior cycle in milestone 2.
2. Build the runnable artifact with a focused packaging command that skips tests after the relevant suite is green, then create a temporary empty project directory with `mktemp -d`.
3. Execute `java -jar target/seed4j-cli.jar apply-set init maven-java --project-path <temporary-project> --project-name 'Sample application' --base-name sampleApplication --node-package-manager npm --package-name com.mycompany.sample --plan`, adapting only the actual JAR filename discovered in `target/`.
4. Assert exit code `0`, output containing `Status: VALID` and `No changes were applied.`, and an empty project directory before removing the temporary directory through an explicit validated path.
5. After the coverage and guard-removal commits, run the focused CLI, catalog, and dependency-planning tests, inspect the four affected classes in the JaCoCo report, apply the post-green design review, then run `./mvnw test`, `npm run prettier:check`, and `habit-hooks --branch`.
6. Do not run `./mvnw clean verify` automatically. Ask the user to run it locally and send the exit code plus any relevant failure summary.

## Documentation Impact

`documentation/Commands.md` is the canonical user-facing command reference. Milestone 2 adds the rule that a relevant project-history type mismatch invalidates the plan and names the explicit override as the safe next action. Milestones 1 and 3 preserve already documented behavior and therefore do not change this file.

`documentation/hexagonal-architecture.md` is the canonical boundary description. Milestone 2 documents that Picocli, Seed4J catalog metadata, and Seed4J project history are converted into typed module-set values or structured unsupported-history facts at their primary/secondary boundaries. Milestones 1 and 3 only make those responsibilities explicit in code.

Completion output and `--help` remain unchanged. No other canonical document describes module-set history type mismatch or these boundary conversions. Milestone 5 requires no documentation edit because it proves already documented conversion behavior and removes only an internal condition made redundant by the execution-order construction invariant.

## Rollout and Recovery

Each planned concern is committed independently and never amended. Recovery consists of reverting the affected commit in reverse dependency order; milestone 3 depends on the typed model from milestone 2, which depends on the structured facts from milestone 1. Milestone 5's test and refactor commits can be reverted independently because neither changes a public contract. No configuration, persisted-history, or Seed4J core migration is introduced because the history adapter interprets existing scalar values at read time.

## Lessons Learned

- The prior review correctly identified that presentation-formatted problems and `Object` values could not be repaired honestly through private helpers; the user explicitly authorized evolving those Java contracts.
- The focused baseline passes, so every intended CLI-preserving refactor has a green comparison point before the typed redesign begins.
- Milestone 1 can derive validity from dependency validations and missing-required details without changing renderer output because those generic problem variants were never rendered; only their duplicated invalidity flag was used.
- Milestone 2 cycle 1 was RED at compilation for the missing integer value variant, then GREEN after typed values and boundary conversion replaced raw planning values.
- The relevant string-history/integer-property cycle was RED because the command returned `0`; the typed mismatch fact and renderer made it GREEN with exit code `2`, no default fallback, and the exact override diagnostic.
- Existing explicit precedence and selected-definition filtering already produced the intended override and irrelevant-history behavior once values were typed; dedicated public-command tests now protect both paths.
- Unsupported history was RED at compilation until its structured fact became part of `ModuleSetPlanningHistory`; a public CLI test now reaches the real Seed4J history adapter with a collection value and observes the actionable invalid plan.
- Multiple mismatches are sorted by domain key before rendering, so their order is stable even when property declaration and history-map order differ.
- Retaining the catalog literal beside the typed default preserves an `INTEGER` conflict between `"02"` and `"2"` even though both resolve to the integer value `2`.
- Milestone 3 preserved the focused public behavior while replacing the dense planner signature with ordered selected modules, explicit values, and history parameter facts; extracted policies remain protected through service and command behavior rather than topology tests.
- The post-green review found no defect or demonstrated structural risk in the changed data flow. The two plausible cleanup candidates were classified `No action`: global option metadata follows different semantics, and duplicate recognized/unsupported facts cannot be produced by the runtime adapter.
- Habit Hooks supplied additional evidence after that initial review: parameter outcome aggregation and precedence stages were sufficiently independent to name without topology tests. Focused tests stayed green and file-scoped Habit Hooks no longer reports enforced findings for the three affected policy files.
- The final branch-wide Habit Hooks run exits `1` with 3 high-parameter-count, 40 oversized-function, and 7 oversized-file issues. The redesign's production planner, resolver, history reader, plan renderer, problem renderer, and top-level planner all pass file-scoped enforced checks; remaining current-file reports are behavior-test size heuristics classified `No action`, while all other enforced findings are outside this task's commit range.
- A confirming `habit-hooks --last 7` scan narrows this task's enforced output to 18 long Given/When/Then test methods and the two pre-existing broad behavior-suite files. It reports no enforced production finding in the seven delivered commits; those test findings remain `No action` because the repository explicitly favors inline setup and assertions over helper extraction, and broad suite splitting is unrelated to the typed redesign.
- The three coverage additions were already supported by production behavior: complex and `null` history values both rendered the structured unsupported-type diagnostic, boolean history rendered its recognized type in an integer mismatch, and strict catalog boolean conversion accepted only `"true"` and `"false"`. Their focused public suites passed without production changes.
- The milestone 5 design review found no defect, design risk, or maintainability opportunity that justified another edit. The boundary mappings are explicit and exhaustive, no test seam entered production, and the dependency-position simplification removes rather than hides an impossible state.
- `./mvnw test` passed all 534 tests. The current unit JaCoCo report records zero missed lines and branches for the four targeted classes: `ModuleSetDependencyPlanner` (0 of 2 branches), `ModuleSetHistoryParameterValueType` (0 of 3), `ProjectsModuleSetPlanningHistoryReader` (0 of 9), and `Seed4JModuleSetCatalog` (0 of 13).
- `npm run prettier:check` passed. `habit-hooks --branch` and the confirming `habit-hooks --last 2` exit `1` for the known branch baseline: production findings outside this milestone, the pre-existing oversized dependency-planning method, broad behavior-suite files, and inline Given/When/Then size heuristics. The new `Seed4JModuleSetCatalogTest` passes its file-scoped Habit Hooks check; no new actionable production finding was introduced, and extracting the reported command-test setup/assertions would conflict with the repository rule favoring explicit public behavior scenarios.
