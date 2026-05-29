# Move Spring Local Runner to Secondary

This ExecPlan is a living document. Keep `Progress`, `Decisions`, `Risks`, and `Lessons Learned` up to date as work advances.

Safety boundary: This task is limited to authorized, defensive maintenance of this repository. Do not provide offensive guidance or policy-bypassing instructions.

## Purpose / Big Picture

The bootstrap domain currently contains Spring Boot technical contracts and composition contains Spring-specific adapters. This refactor moves Spring local-runner implementation details to `bootstrap/infrastructure/secondary` while preserving runtime behavior. After completion, the domain keeps only the local-runner port and does not know Spring types for local CLI execution.

## Scope

In scope:

- Move local Spring execution implementation from domain/composition into secondary classes:
  - `SpringBootLocalCliRunner`
  - `SpringApplicationBuilderAdapter`
  - `SpringApplicationContextAdapter`
  - `SpringBootExitCodeResolver`
- Update wiring and launcher factory dependencies to consume the new secondary runner.
- Preserve existing behavior and tests through equivalent or improved coverage.

Out of scope:

- Redesigning `PreSpringBootstrapApplicationService`.
- Moving `JavaProcessChildLauncher` or `ChildProcessLauncher` out of domain.
- Runtime-mode protocol redesign.

## Definitions

`LocalCliRunner`: domain port that executes CLI in-process.

`Secondary`: technical adapters package `com.seed4j.cli.bootstrap.infrastructure.secondary`.

`Composition`: wiring module `com.seed4j.cli.bootstrap.composition` that instantiates concrete dependencies.

## Existing Context

- `LocalSpringCliRunner` in `bootstrap/domain` exposes Spring technical interfaces and types.
- `PreSpringBootstrapConfiguration` contains Spring adapter records/methods for builder/context/exit-code.
- `Seed4JCliLauncherFactory` builds `LocalSpringCliRunner` from Spring-typed dependencies.

## Desired End State

- `LocalCliRunner` is the only local execution contract in domain and is `public`.
- Spring technical implementation lives under secondary with the four named classes.
- `PreSpringBootstrapConfiguration` no longer declares Spring adapter implementations.
- `Seed4JCliLauncherFactory` no longer depends on `LocalSpringCliRunner` technical inner contracts.
- Behavior remains unchanged for config path loading, extension start-class handling, and Spring bootstrap settings.

## Milestones

### Milestone 1 - Introduce secondary Spring runner with behavior parity

#### Goal

Add `SpringBootLocalCliRunner` and collaborators in secondary with equivalent behavior to current `LocalSpringCliRunner`.

#### Changes

- [ ] Make `LocalCliRunner` public in domain.
- [ ] Add `SpringBootLocalCliRunner` in secondary.
- [ ] Add `SpringApplicationBuilderAdapter`, `SpringApplicationContextAdapter`, and `SpringBootExitCodeResolver` in secondary.
- [ ] Add/adjust tests that prove config path, extension start-class, and Spring runtime flags behavior.

#### Validation

- [ ] Command: `./mvnw -Dtest=SpringBootLocalCliRunnerTest test`
- [ ] Expected result: all local-runner behavior tests pass.

#### Acceptance Criteria

- [ ] Secondary runner behavior matches current local Spring runner behavior.
- [ ] New runner compiles and runs through `LocalCliRunner`.

### Milestone 2 - Rewire launcher factory and composition to new secondary runner

#### Goal

Replace domain/composition Spring adapters with the new secondary classes in production wiring.

#### Changes

- [ ] Update `Seed4JCliLauncherFactory.LauncherDependencies` to receive `LocalCliRunner`.
- [ ] Update factory creation path to use provided `LocalCliRunner` directly.
- [ ] Update `PreSpringBootstrapConfiguration` to instantiate `SpringBootLocalCliRunner`.
- [ ] Remove composition-local Spring adapter records/methods.
- [ ] Update affected tests.

#### Validation

- [ ] Command: `./mvnw -Dtest=Seed4JCliLauncherFactoryTest,PreSpringBootstrapConfigurationTest test`
- [ ] Expected result: wiring/factory tests pass with new dependencies.

#### Acceptance Criteria

- [ ] Composition no longer contains Spring adapter implementation details.
- [ ] Launcher factory no longer references `LocalSpringCliRunner` contracts.

### Milestone 3 - Remove legacy domain runner and finalize verification

#### Goal

Delete legacy domain implementation and validate broad bootstrap coverage.

#### Changes

- [ ] Remove `LocalSpringCliRunner` from domain.
- [ ] Replace or relocate old tests to secondary package.
- [ ] Update in-process bootstrap tests to use secondary runner.
- [ ] Confirm no Spring local-runner adapter types remain in domain/composition.

#### Validation

- [ ] Command: `rg -n "LocalSpringCliRunner|ApplicationBuilder|ApplicationContext" src/main/java/com/seed4j/cli/bootstrap/domain src/main/java/com/seed4j/cli/bootstrap/composition`
- [ ] Expected result: no remaining references to removed legacy runner contracts.
- [ ] Command: `./mvnw -Dtest=SpringBootLocalCliRunnerTest,Seed4JCliLauncherFactoryTest,PreSpringBootstrapConfigurationTest,ExtensionRuntimeBootstrapInProcessTest,Seed4JCliLauncherTest test`
- [ ] Expected result: targeted bootstrap suite passes.
- [ ] Command: `./mvnw clean verify`
- [ ] Expected result: full project verification passes.

#### Acceptance Criteria

- [ ] Legacy domain runner removed.
- [ ] Secondary owns local Spring execution implementation.
- [ ] Full verification green.

## Progress

- [x] Milestone 1 started
- [x] Milestone 1 completed
- [x] Milestone 2 started
- [x] Milestone 2 completed
- [x] Milestone 3 started
- [x] Milestone 3 completed

## Decisions

- Decision: Keep this refactor scoped to local Spring runner relocation only.
  Rationale: User requested focusing on this architectural slice and postponing adjacent debts.
  Date/Author: 2026-05-29 / Codex

- Decision: Keep a technical secondary abstraction (`SpringApplicationBuilderOperations`) inside infrastructure.
  Rationale: It enables fast deterministic unit tests for `SpringBootLocalCliRunner` without reintroducing Spring types into domain.
  Date/Author: 2026-05-29 / Codex

- Decision: Update in-process extension bootstrap tests to use `SpringBootLocalCliRunner` directly.
  Rationale: These tests validate public bootstrap behavior and no longer need local Spring adapters in test code.
  Date/Author: 2026-05-29 / Codex

## Risks and Mitigations

- Risk: Behavior drift in local runtime flags (`bannerMode`, `web`, `lazyInitialization`).
  Mitigation: Preserve existing assertions and equivalent tests in new secondary test class.

- Risk: Config precedence regressions for external config and extension start class.
  Mitigation: Port old behavior tests and keep exact property values asserted.

- Risk: Hidden coupling in in-process extension tests to removed domain types.
  Mitigation: Updated local runner construction path and preserved all extension-mode assertions; targeted and full suites passed.

## Validation Strategy

1. Run focused local-runner tests first.
2. Run launcher factory and composition tests after rewiring.
3. Run in-process bootstrap and launcher behavior suite.
4. Run full `./mvnw clean verify`.

## Rollout and Recovery

This is internal refactoring with no expected CLI contract change. Rollout is merge after full verification succeeds. Recovery is a single revert commit of this refactor if regressions appear.

## Lessons Learned

- The domain class `LocalSpringCliRunner` was not needed after moving Spring concerns to secondary; deleting it reduced surface area cleanly.
- `PreSpringBootstrapConfiguration` became materially simpler after replacing inline Spring adapters with `SpringBootLocalCliRunner`.
- Full `clean verify` remained green after the refactor, including integration tests and coverage gates.
