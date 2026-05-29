# Refatorar bootstrap pré-Spring e runtime extension install application service

This ExecPlan is a living document. Keep `Progress`, `Decisions`, `Risks`, and `Lessons Learned` up to date as work advances.

Last repository refresh: 2026-05-29.

Safety boundary: This task is limited to authorized, defensive maintenance of this repository. Do not provide offensive guidance or policy-bypassing instructions.

## Purpose / Big Picture

O objetivo original era separar orquestração pré-Spring de composição técnica e, em seguida, mover a orquestração de runtime extension para application. O repositório mudou desde a primeira versão deste plano: a parte pré-Spring já foi refatorada novamente e agora usa `PreSpringBootstrapRunner` como adapter primário, `PreSpringBootstrapApplicationService` como orquestrador e `PreSpringBootstrapConfiguration` como composição manual.

O objetivo atualizado é preservar essa arquitetura pré-Spring atual, sem reintroduzir o antigo `PreSpringLauncherAssembler`, e concluir o que ainda está pendente para o refactor: criar um `RuntimeExtensionApplicationService` focado em `install`, com wiring Spring explícito para que comandos pós-Spring não instanciem adapters de filesystem diretamente.

Para o usuário final, o comportamento deve continuar igual no startup do CLI e em `seed4j extension install`. Métodos de application service para `enable` e `disable` ficam fora deste refactor porque pertencem à implementação futura da feature de habilitar/desabilitar runtime extension.

## Scope

In-scope:

- Tratar a estrutura pré-Spring atual como baseline: `Seed4JCliApp -> PreSpringBootstrapRunner -> PreSpringBootstrapApplicationService -> PreSpringLauncher`.
- Manter `PreSpringBootstrapConfiguration` em `bootstrap/composition` como wiring manual pré-Spring.
- Manter `PreSpringBootstrapRunner` em `bootstrap/infrastructure/primary` como adapter primário.
- Manter `Seed4JCliLauncherFactory` no domínio sem import de `bootstrap/infrastructure/secondary`.
- Introduzir `bootstrap/application/RuntimeExtensionApplicationService` para `install`.
- Fazer o application service receber portas (`RuntimeModeConfigurationRepository`, `RuntimeExtensionArtifactsRepository`) e `Path userHome`, criando internamente os serviços de domínio.
- Criar wiring Spring explícito para runtime extension, preferencialmente em `bootstrap/composition`, para expor beans de portas e application service.
- Refatorar `ExtensionInstallCommand` para depender do application service.
- Atualizar testes e fixtures que ainda instanciam `ExtensionInstallCommand` com `String userHomePath`.

Out-of-scope:

- Reintroduzir `PreSpringLauncherAssembler`; esse nome está obsoleto no repositório atual.
- Alterar contrato funcional de runtime mode no launcher.
- Introduzir bypass de validação no bootstrap.
- Criar os comandos públicos `seed4j extension enable` e `seed4j extension disable`; isso permanece no ExecPlan dedicado de enable/disable.
- Adicionar métodos `Path enable()` e `Path disable()` ao `RuntimeExtensionApplicationService`; isso pertence à feature futura de enable/disable.
- Orquestrar `RuntimeExtensionModeEnabler` ou `RuntimeExtensionModeDisabler` na camada application neste refactor.
- Redesenhar comandos não relacionados em `command/infrastructure/primary`.
- Mudar formato do arquivo `~/.config/seed4j-cli/config.yml`.
- Mover `PreSpringRuntimeEnvironment` ou `PreSpringRuntimeEnvironmentReader` de pacote neste plano.

## Definitions

- Pré-Spring: fluxo executado em `main(...)` antes de `SpringApplicationBuilder.run(...)`.
- Primary adapter: adapter que recebe entrada externa e dirige o caso de uso. No baseline atual: `PreSpringBootstrapRunner`.
- Composition: pacote responsável por montar objetos concretos. No baseline atual: `bootstrap/composition/PreSpringBootstrapConfiguration`.
- Porta: interface usada para inversão de dependência, como `RuntimeModeConfigurationRepository` e `RuntimeExtensionArtifactsRepository`.
- Adapter secondary: implementação técnica de porta, como `FileSystemRuntimeModeConfigurationRepository` e `FileSystemRuntimeExtensionArtifactsRepository`.
- Application service de runtime extension install: serviço da camada de aplicação que orquestra `install` criando o serviço de domínio internamente.

## Current Repository Snapshot

Estado atualizado em 2026-05-29:

- `src/main/java/com/seed4j/cli/Seed4JCliApp.java` expõe `productionBootstrapExitCodeResolver()` e delega para `PreSpringBootstrapConfiguration.preSpringBootstrapRunner()::exitCodeFor`.
- `src/main/java/com/seed4j/cli/bootstrap/infrastructure/primary/PreSpringBootstrapRunner.java` recebe `String[] args`, cria `PreSpringBootstrapCommand` e delega para `PreSpringBootstrapApplicationService`.
- `src/main/java/com/seed4j/cli/bootstrap/application/PreSpringBootstrapApplicationService.java` lê o ambiente por `PreSpringRuntimeEnvironmentReader`, cria um `PreSpringLauncher` via `PreSpringLauncherFactory` e executa `launch(args, childMode)`.
- `PreSpringRuntimeEnvironment` e `PreSpringRuntimeEnvironmentReader` estão atualmente em `bootstrap/domain`.
- `src/main/java/com/seed4j/cli/bootstrap/composition/PreSpringBootstrapConfiguration.java` monta `CurrentProcessPreSpringRuntimeEnvironmentReader`, `SpringBootLocalCliRunner`, `JavaChildProcessCommandExecutor`, `FileSystemRuntimeModeConfigurationRepository`, `Seed4JCliLauncherFactory` e `PreSpringBootstrapRunner`.
- `src/main/java/com/seed4j/cli/bootstrap/domain/Seed4JCliLauncherFactory.java` já recebe `RuntimeModeConfigurationRepository` por parâmetro e não importa adapters secondary.
- `src/main/java/com/seed4j/cli/command/infrastructure/primary/ExtensionInstallCommand.java` recebe `RuntimeExtensionApplicationService` por construtor e não instancia adapters de filesystem.
- `RuntimeExtensionInstaller`, `RuntimeExtensionModeEnabler` e `RuntimeExtensionModeDisabler` já existem em `bootstrap/domain`.
- `src/main/java/com/seed4j/cli/bootstrap/application/RuntimeExtensionApplicationService.java` existe em `bootstrap/application`, recebe `Path userHome` e portas no construtor, e delega `install(...)` para `RuntimeExtensionInstaller`.
- `src/main/java/com/seed4j/cli/bootstrap/composition/RuntimeExtensionApplicationConfiguration.java` expõe beans de `RuntimeModeConfigurationRepository`, `RuntimeExtensionArtifactsRepository` e `RuntimeExtensionApplicationService`.
- Ainda não existem `ExtensionEnableCommand` nem `ExtensionDisableCommand` no código atual.

Premissas obsoletas da versão anterior do plano:

- `PreSpringLauncherAssembler` não existe mais e não deve ser recriado.
- `Seed4JCliApp.productionBootstrapEntryPoint(...)` não é o nome atual; o método relevante é `productionBootstrapExitCodeResolver()`.
- `BootstrapRuntimeConfiguration` em `bootstrap/infrastructure/secondary` não combina mais com a direção atual; wiring deve ficar em `bootstrap/composition` ou em uma configuração Spring claramente de composição.

## Desired End State

- A trilha pré-Spring atual permanece estável: `Seed4JCliApp -> PreSpringBootstrapRunner -> PreSpringBootstrapApplicationService`, com wiring em `PreSpringBootstrapConfiguration`.
- `PreSpringBootstrapConfiguration` continua sendo composição manual e não vira canal de execução de caso de uso.
- `Seed4JCliLauncherFactory` continua sem dependência para `bootstrap/infrastructure/secondary`.
- `RuntimeExtensionApplicationService` existe em `bootstrap/application` com método para `install`.
- `RuntimeExtensionApplicationService` recebe `Path userHome`, `RuntimeModeConfigurationRepository` e `RuntimeExtensionArtifactsRepository`, e instancia internamente `RuntimeExtensionInstaller`.
- O wiring Spring cria beans concretos para as portas e para o application service.
- `ExtensionInstallCommand` depende de `RuntimeExtensionApplicationService` e não instancia adapters de filesystem.
- Comportamento observável de `seed4j extension install` permanece compatível, incluindo mensagens e exit codes.

## Milestones

### Milestone 0 - Atualizar baseline do ExecPlan

#### Goal

Sincronizar este plano com o repositório atual antes de qualquer nova alteração de código.

#### Changes

- [x] Atualizar contexto pré-Spring para `PreSpringBootstrapRunner`, `PreSpringBootstrapApplicationService` e `PreSpringBootstrapConfiguration`.
- [x] Marcar `PreSpringLauncherAssembler` como premissa obsoleta.
- [x] Registrar que `ExtensionInstallCommand` ainda faz composição manual de adapters.
- [x] Registrar que serviços de domínio de `install`, `enable` e `disable` já existem, mas que apenas `install` entra neste refactor.

#### Validation

Command: `rg --files src/main/java src/test/java documentation .agent | rg '(bootstrap|extension|runtime|pre-spring|assembler|application-service|Runtime|Extension|Bootstrap)'`
Expected result: arquivos atuais de bootstrap/runtime extension identificados.

Command: leituras focadas de `Seed4JCliApp`, `PreSpringBootstrapConfiguration`, `PreSpringBootstrapApplicationService`, `PreSpringBootstrapRunner`, `Seed4JCliLauncherFactory` e `ExtensionInstallCommand`.
Expected result: baseline documentado corresponde ao código atual.

#### Acceptance Criteria

- [x] O plano não aponta mais para criação de `PreSpringLauncherAssembler`.
- [x] O plano diferencia claramente o trabalho pré-Spring já feito do trabalho pós-Spring pendente.

### Milestone 1 - Auditar fronteira pré-Spring atual

#### Goal

Confirmar que a arquitetura pré-Spring atual continua preservada enquanto o refactor de runtime extension avança.

#### Changes

Editar somente se a auditoria encontrar drift:

1. `src/main/java/com/seed4j/cli/Seed4JCliApp.java` deve continuar delegando para o primary pré-Spring produzido por composition.
2. `src/main/java/com/seed4j/cli/bootstrap/infrastructure/primary/PreSpringBootstrapRunner.java` deve continuar apenas adaptando `String[] args` para `PreSpringBootstrapCommand`.
3. `src/main/java/com/seed4j/cli/bootstrap/application/PreSpringBootstrapApplicationService.java` deve continuar orquestrando o caso de uso, sem instanciar adapters secondary diretamente.
4. `src/main/java/com/seed4j/cli/bootstrap/composition/PreSpringBootstrapConfiguration.java` deve continuar apenas montando dependências concretas.
5. `src/main/java/com/seed4j/cli/bootstrap/domain/Seed4JCliLauncherFactory.java` deve continuar sem import de secondary.

#### Validation

Command: `./mvnw -Dtest=Seed4JCliAppTest,PreSpringBootstrapApplicationServiceTest,PreSpringBootstrapRunnerTest,PreSpringBootstrapConfigurationTest,Seed4JCliLauncherFactoryTest,Seed4JCliLauncherTest test`
Expected result: testes focados de bootstrap e launcher passam.

Command: `rg -n "PreSpringLauncherAssembler|bootstrap\.infrastructure\.secondary" src/main/java/com/seed4j/cli/bootstrap/domain src/main/java/com/seed4j/cli/bootstrap/application src/main/java/com/seed4j/cli/Seed4JCliApp.java`
Expected result: nenhum match para `PreSpringLauncherAssembler`; nenhum import de secondary em domain/application/app.

#### Acceptance Criteria

1. `Seed4JCliApp` não contém composição detalhada de launcher.
2. `PreSpringBootstrapRunner` permanece como adapter primário.
3. `PreSpringBootstrapConfiguration` permanece composição manual.
4. Não há dependência `bootstrap/domain -> bootstrap/infrastructure/secondary`.

### Milestone 2 - Introduzir application service para runtime extension install

#### Goal

Concentrar a orquestração de `install` na camada `application`, recebendo portas e valores técnicos, sem misturar comando CLI com composição técnica.

#### Changes

Editar/criar os seguintes arquivos:

1. Criar `src/main/java/com/seed4j/cli/bootstrap/application/RuntimeExtensionApplicationService.java` com:
   - construtor recebendo `Path userHome`, `RuntimeModeConfigurationRepository`, `RuntimeExtensionArtifactsRepository`;
   - método `RuntimeExtensionInstallResult install(RuntimeExtensionInstallRequest request)`.
2. O service deve criar internamente:
   - `RuntimeExtensionInstaller` para `install`.
3. Criar `src/test/java/com/seed4j/cli/bootstrap/application/RuntimeExtensionApplicationServiceTest.java` cobrindo delegação e wiring de portas sem repetir todas as regras já cobertas nos testes de domínio.
4. Criar configuração Spring em `src/main/java/com/seed4j/cli/bootstrap/composition/RuntimeExtensionApplicationConfiguration.java` ou nome equivalente alinhado ao projeto, com beans para:
   - `Path userHome` ou factory interna baseada em `@Value("${user.home}")`;
   - `RuntimeModeConfigurationRepository`;
   - `RuntimeExtensionArtifactsRepository`;
   - `RuntimeExtensionApplicationService`.
5. Manter adapters secondary livres de `@Autowired`; usar construtor/beans explícitos.

#### Validation

Command: `./mvnw -Dtest=RuntimeExtensionApplicationServiceTest,RuntimeExtensionInstallerTest test`
Expected result: service de aplicação delega corretamente e regras de domínio continuam cobertas pelos testes existentes.

#### Acceptance Criteria

1. `RuntimeExtensionApplicationService` está em `bootstrap/application`.
2. O service recebe portas/interfaces, não adapters concretos.
3. O service não contém regra de domínio complexa; ele apenas orquestra criação e chamada do serviço de domínio de install.
4. `RuntimeExtensionApplicationService` não expõe `enable()` nem `disable()`.

### Milestone 3 - Refatorar comando `extension install` para usar application service

#### Goal

Remover composição técnica do adapter primário `ExtensionInstallCommand` preservando comportamento observável.

#### Changes

Editar os seguintes arquivos:

1. `src/main/java/com/seed4j/cli/command/infrastructure/primary/ExtensionInstallCommand.java`:
   - substituir construtor com `@Value("${user.home}") String userHomePath` por construtor recebendo `RuntimeExtensionApplicationService`;
   - trocar chamada direta a `RuntimeExtensionInstaller.install(...)` por `RuntimeExtensionApplicationService.install(...)`;
   - manter tratamento de `InvalidRuntimeConfigurationException`, mensagens de sucesso e exit codes.
2. `src/test/java/com/seed4j/cli/command/infrastructure/primary/ExtensionInstallCommandTest.java`:
   - atualizar construção do comando para usar application service real ou test double explícito;
   - manter assertions de saída e persistência já existentes.
3. `src/test/java/com/seed4j/cli/command/infrastructure/primary/CliFixture.java`:
   - atualizar montagem manual para criar `RuntimeExtensionApplicationService` em vez de chamar o construtor antigo do comando.
4. `src/test/java/com/seed4j/cli/command/infrastructure/primary/Seed4JCommandsFactoryTest.java`:
   - ajustar apenas se a árvore de comandos ou wiring do fixture mudar.

#### Validation

Command: `./mvnw -Dtest=ExtensionInstallCommandTest,Seed4JCommandsFactoryTest test`
Expected result: help de `extension` e comportamento de `install` continuam iguais.

Command: `rg -n "new FileSystemRuntimeModeConfigurationRepository|new FileSystemRuntimeExtensionArtifactsRepository|new RuntimeExtensionInstaller" src/main/java/com/seed4j/cli/command/infrastructure/primary`
Expected result: nenhum match em comandos primários.

#### Acceptance Criteria

1. `ExtensionInstallCommand` não contém `new FileSystemRuntimeModeConfigurationRepository(...)`.
2. `ExtensionInstallCommand` não contém `new FileSystemRuntimeExtensionArtifactsRepository()`.
3. `ExtensionInstallCommand` não contém `new RuntimeExtensionInstaller(...)`.
4. Mensagens atuais de sucesso e erro de `extension install` permanecem iguais.
5. Exit code de sucesso continua `0`; erro de configuração/runtime continua `ExitCode.SOFTWARE`.

### Milestone 4 - Consolidação arquitetural e validação ampla

#### Goal

Fechar o refactor com validação ampla e atualizar este plano para handoff.

#### Changes

Editar os seguintes arquivos conforme necessário:

1. Atualizar este ExecPlan com estado final de progresso, decisões, riscos e aprendizados.
2. Atualizar documentação arquitetural somente se o comportamento público ou a fronteira documentada mudar:
   - `documentation/Commands.md`;
   - `README.md`.
3. Revisar imports e dependências para garantir ausência de novos vazamentos de camada.

#### Validation

Command: `./mvnw clean verify`
Expected result: suíte completa, cobertura, checkstyle e integração passam.

Command: `npm run prettier:check`
Expected result: sem divergências de formatação.

Command: `seed4j --version`
Expected result: comando executa com exit code `0` e saída de versão compatível.

#### Acceptance Criteria

1. Pipeline local completo passa.
2. Não há dependência `domain/application/command primary -> concrete filesystem adapters`, exceto via composition/configuração adequada.
3. Documento de execução reflete exatamente o estado final implementado.

## Progress

- [x] ExecPlan drafted
- [x] Milestone 1 original completed before repository refresh
- [x] Repository baseline refreshed on 2026-05-29
- [x] Milestone 0 completed
- [x] Milestone 1 audit started
- [x] Milestone 1 audit completed
- [x] Milestone 2 started
- [x] Milestone 2 completed
- [x] Milestone 3 started
- [x] Milestone 3 completed
- [x] Milestone 4 started
- [x] Milestone 4 completed

Execução desta rodada (TDD estrito autônomo):

- `Cycle 1 | Introduzir RuntimeExtensionApplicationService.install delegando via portas | expected failure: compilação por classe ausente | 🔴 red result: RuntimeExtensionApplicationService not found | 🌱 green change: criar bootstrap/application/RuntimeExtensionApplicationService | suite result: RuntimeExtensionApplicationServiceTest + RuntimeExtensionInstallerTest verdes | vertical checkpoint: n/a | 🌀 refactor: sem mudanças`
- `Cycle 2 | Introduzir wiring Spring explícito de runtime extension | expected failure: compilação por configuração ausente | 🔴 red result: RuntimeExtensionApplicationConfiguration not found | 🌱 green change: criar bootstrap/composition/RuntimeExtensionApplicationConfiguration | suite result: testes de configuração/aplicação/domínio verdes | vertical checkpoint: ExtensionInstallCommandTest verde | 🌀 refactor: sem mudanças`
- `Cycle 3 | Migrar ExtensionInstallCommand para depender de RuntimeExtensionApplicationService | expected failure: incompatibilidade de construtor | 🔴 red result: compile error em ExtensionInstallCommandTest | 🌱 green change: refatorar ExtensionInstallCommand + CliFixture para service injection | suite result: ExtensionInstallCommandTest + Seed4JCommandsFactoryTest verdes | vertical checkpoint: incluso na suíte do ciclo | 🌀 refactor: sem mudanças`
- `Cycle 4 | Completar comportamentos pendentes dos novos testes de application/composition | expected failure: n/a (fechamento de cobertura comportamental) | 🔴 red result: n/a | 🌱 green change: adicionar testes de propagação de erro e escopo por user home | suite result: suíte focada combinada verde | vertical checkpoint: incluso na suíte do ciclo | 🌀 refactor: formatar arquivos com Prettier`

Validação final da rodada:

- `./mvnw clean verify`: verde (na segunda execução; primeira execução falhou de forma transitória com `ClassNotFoundException` do surefire para `RuntimeExtensionApplicationServiceTest` após `clean`, sem reproduzir no rerun).
- `npm run prettier:check`: verde.
- `seed4j --version`: exit code `0`; saída `Seed4J CLI v0.0.1-SNAPSHOT` e `Seed4J version: 1.34.0`.

Historical execution notes from the original Milestone 1:

- `PreSpringBootstrapApplicationService` was introduced in `bootstrap/application`.
- Direct `domain -> infrastructure.secondary` dependency was removed from `Seed4JCliLauncherFactory`.
- A later refactor replaced the old assembler language with the current `PreSpringBootstrapRunner` + `PreSpringBootstrapConfiguration` architecture.

Repository refresh notes (2026-05-29):

- Current pre-Spring composition class is `PreSpringBootstrapConfiguration`; there is no `PreSpringLauncherAssembler`.
- Current primary adapter is `PreSpringBootstrapRunner`.
- Current app entry method is `productionBootstrapExitCodeResolver()`.
- Runtime extension domain services for `install`, `enable` and `disable` exist, but only `install` is in scope for this refactor.
- Runtime extension install application service e wiring Spring explícito foram implementados.
- `ExtensionInstallCommand` foi migrado para depender de `RuntimeExtensionApplicationService`, removendo composição técnica do adapter primário.

## Decisions

- Decision: Do not reintroduce `PreSpringLauncherAssembler`.
  Rationale: the repository now models pre-Spring wiring through `bootstrap/composition/PreSpringBootstrapConfiguration` and execution through `PreSpringBootstrapRunner`; recreating the assembler would move the design backwards.
  Date/Author: 2026-05-29 / Renan + Codex

- Decision: Keep `PreSpringBootstrapRunner` as the primary adapter.
  Rationale: `Seed4JCliApp` should remain a thin client of bootstrap and should not execute application logic or detailed composition.
  Date/Author: 2026-05-29 / Renan + Codex

- Decision: Keep pre-Spring manual wiring in `bootstrap/composition`.
  Rationale: Spring is not available before bootstrap, so this package replaces container wiring without becoming an application service.
  Date/Author: 2026-05-29 / Renan + Codex

- Decision: `RuntimeExtensionApplicationService` exposes only `install` in this refactor.
  Rationale: adding `enable()` and `disable()` would implement part of the future enable/disable feature, so those methods should be introduced when that feature is actually executed.
  Date/Author: 2026-05-29 / Renan + Codex

- Decision: Application service receives ports/interfaces, not concrete filesystem adapters.
  Rationale: keeps adapter selection in composition/Spring wiring and avoids primary adapters doing technical composition.
  Date/Author: 2026-05-27, refreshed 2026-05-29 / Renan + Codex

- Decision: Do not include public `extension enable`/`extension disable` commands or application methods in this ExecPlan.
  Rationale: another ExecPlan owns the feature; this plan should only clean up the current `extension install` composition leak.
  Date/Author: 2026-05-29 / Renan + Codex

## Risks and Mitigations

- Risk: accidentally reintroducing old assembler terminology and duplicating the current composition class.
  Mitigation: treat `PreSpringBootstrapConfiguration` as the canonical pre-Spring wiring point.

- Risk: `RuntimeExtensionApplicationService` becoming a domain rules dump.
  Mitigation: keep domain behavior inside `RuntimeExtensionInstaller`; do not add `enable()` or `disable()` until the corresponding feature is executed.

- Risk: Spring bean wiring introduces ambiguous `Path` beans or unclear `userHome` ownership.
  Mitigation: avoid broad generic `Path` beans if possible; create service through an explicit `@Bean` method receiving `@Value("${user.home}") String userHomePath`.

- Risk: changes in `ExtensionInstallCommand` alter messages expected by tests/scripts.
  Mitigation: preserve current output strings and validate with `ExtensionInstallCommandTest`.

- Risk: command tests become too mocked and stop covering filesystem behavior.
  Mitigation: keep at least one `ExtensionInstallCommandTest` path using real filesystem adapters through the application service.

## Validation Strategy

1. Run focused bootstrap audit tests after any pre-Spring touch.
2. Run `RuntimeExtensionApplicationServiceTest` plus existing install domain tests after introducing the application service.
3. Run `ExtensionInstallCommandTest` and `Seed4JCommandsFactoryTest` after command migration.
4. Run `./mvnw clean verify`.
5. Run `npm run prettier:check`.
6. Confirm manual minimum:
   - `seed4j --version` continues working;
   - `seed4j extension install ...` keeps current success/error behavior when a valid test runtime extension jar is available.

## Rollout and Recovery

Rollout:

1. Treat Milestone 2 as internal architecture refactor without public CLI behavior changes.
2. Treat Milestone 3 as behavior-preserving command migration.
3. Merge only after focused and full validation pass.

Recovery:

1. If startup regresses, revert only changes touching pre-Spring baseline; this plan should avoid such changes unless audit finds drift.
2. If `extension install` regresses, revert Milestone 3 while keeping `RuntimeExtensionApplicationService` if it is independently green.
3. If Spring wiring fails, temporarily instantiate the application service in `CliFixture`/tests while fixing production configuration, but do not restore adapter construction inside `ExtensionInstallCommand` as the long-term state.
4. Revalidate with focused tests and `./mvnw clean verify` after rollback.

## Lessons Learned

- The current repository uses `composition` as the pre-Spring replacement for container wiring; using an `Assembler` name here is now misleading.
- Keeping `PreSpringBootstrapRunner` explicit makes the `Seed4JCliApp` boundary clearer than calling application services directly from the app class.
- The next valuable cleanup is no longer in launcher creation; it is in moving runtime extension command orchestration out of primary adapters.
- A runtime extension application service should grow only with the feature currently being implemented; exposing `enable()`/`disable()` before the CLI feature would blur refactor and feature scope.
