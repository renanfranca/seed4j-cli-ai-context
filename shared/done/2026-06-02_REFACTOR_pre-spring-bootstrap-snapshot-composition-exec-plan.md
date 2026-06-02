# Refatorar bootstrap pré-Spring para composição por snapshot

This ExecPlan is a living document. Keep `Progress`, `Decisions`, `Risks and Mitigations`, and `Lessons Learned` up to date as work advances.

Safety boundary: This task is limited to authorized, defensive maintenance of this repository. Do not provide offensive guidance or policy-bypassing instructions.

## Purpose / Big Picture

O objetivo é preservar o comportamento observável do CLI e simplificar a arquitetura do bootstrap pré-Spring. O usuário final continua vendo o mesmo resultado ao executar `seed4j`, `seed4j --version` e os fluxos de runtime mode, mas a composição manual agora lê o ambiente técnico uma única vez, monta o launcher com esse snapshot e entrega um serviço de aplicação mais direto.

A mudança remove as abstrações intermediárias `PreSpringLauncher` e `PreSpringLauncherFactory`, aproxima `PreSpringBootstrapApplicationService` do estilo de um use case simples e deixa explícito que `childMode` é estado de construção do launcher, não um argumento público do fluxo de aplicação.

## Scope

In scope:

- Manter `PreSpringRuntimeEnvironmentReader` como porta de leitura do ambiente técnico pré-Spring.
- Manter `CurrentProcessPreSpringRuntimeEnvironmentReader` como adapter secondary que lê `System.getProperty(...)`.
- Fazer `PreSpringBootstrapConfiguration` ler um `PreSpringRuntimeEnvironment` uma vez e usar esse snapshot para montar o grafo.
- Alterar `Seed4JCliLauncherFactory` para receber `PreSpringRuntimeEnvironment`.
- Fazer `Seed4JCliLauncher` carregar `childMode` como estado construído pela factory.
- Fazer `PreSpringBootstrapApplicationService.exitCodeFor(...)` chamar apenas `seed4jCliLauncher.launch(command.args())`.
- Remover `PreSpringLauncher` e `PreSpringLauncherFactory`.
- Atualizar os testes de application, composition, primary runner, app main e launcher.

Out of scope:

- Alterar as regras de runtime mode (`standard` vs `extension`).
- Alterar o formato de `~/.config/seed4j-cli/config.yml`.
- Alterar mensagens públicas, opções CLI ou exit codes.
- Mover o local runner Spring Boot ou o repositório de filesystem para outros pacotes.
- Rodar `./mvnw clean verify` automaticamente; o gate final continua com o usuário.

## Definitions

- Pré-Spring: código executado antes de `SpringApplicationBuilder.run(...)`.
- Snapshot de ambiente: instância de `PreSpringRuntimeEnvironment` contendo `userHomePath`, `executablePath`, `childMode` e `javaExecutablePath` lidos uma única vez.
- Porta: interface de fronteira arquitetural. Aqui, `PreSpringRuntimeEnvironmentReader` representa acesso ao ambiente técnico do processo.
- Adapter secondary: implementação técnica chamada pelo bootstrap, como `CurrentProcessPreSpringRuntimeEnvironmentReader`.
- Composition: pacote `bootstrap/composition`, responsável por instanciar e conectar objetos concretos antes do Spring existir.
- Application service: classe de aplicação que executa o caso de uso sem montar adapters concretos.

## Existing Context

O estado anterior do repositório tinha estas características:

- `Seed4JCliApp.productionBootstrapExitCodeResolver()` delegava para `PreSpringBootstrapConfiguration.preSpringBootstrapRunner()::exitCodeFor`.
- `PreSpringBootstrapRunner` era o adapter primário e criava `PreSpringBootstrapCommand`.
- `PreSpringBootstrapApplicationService` ainda recebia `PreSpringLauncherFactory` e `PreSpringRuntimeEnvironmentReader`; no método `exitCodeFor`, ele lia o ambiente, criava um `PreSpringLauncher` e chamava `launch(args, childMode)`.
- `PreSpringLauncher` e `PreSpringLauncherFactory` ficavam em `bootstrap/application`, mas só escondiam o `Seed4JCliLauncher`.
- `PreSpringBootstrapConfiguration` instanciava `CurrentProcessPreSpringRuntimeEnvironmentReader`, `SpringBootLocalCliRunner`, `JavaChildProcessCommandExecutor`, `FileSystemRuntimeModeConfigurationRepository` e `Seed4JCliLauncherFactory`.
- `Seed4JCliLauncher` ficava em `bootstrap/domain` e expunha `launch(String[] args, boolean childMode)`.
- `Seed4JCliLauncherFactory.LauncherDependencies` carregava `javaExecutable`, embora essa informação já existisse em `PreSpringRuntimeEnvironment`.

## Desired End State

A trilha pré-Spring fica:

`Seed4JCliApp -> PreSpringBootstrapConfiguration -> PreSpringBootstrapRunner -> PreSpringBootstrapApplicationService -> Seed4JCliLauncher`

A composition lê o ambiente técnico uma única vez por `PreSpringRuntimeEnvironmentReader`, monta o `Seed4JCliLauncher` com esse snapshot e entrega o launcher pronto para o application service. O application service não conhece reader nem factory intermediária; ele só valida o command e chama `seed4jCliLauncher.launch(command.args())`.

`Seed4JCliLauncher` decide child mode a partir do estado recebido na construção. O método público de execução usado pelo application é `launch(String[] args)`. Não sobra uso de `PreSpringLauncher`, `PreSpringLauncherFactory` ou `launch(args, childMode)`.

## Milestones

### Milestone 1 - Fazer o launcher carregar o snapshot pré-Spring

#### Goal

Mover `childMode` e `javaExecutablePath` para o estado construído pelo `Seed4JCliLauncherFactory`, sem mudar o comportamento de execução.

#### Changes

- `src/main/java/com/seed4j/cli/bootstrap/domain/Seed4JCliLauncherFactory.java` passou a receber `PreSpringRuntimeEnvironment` em `create(...)`.
- `LauncherDependencies` passou a carregar apenas `ProcessCommandExecutor` e `LocalCliRunner`.
- `src/main/java/com/seed4j/cli/bootstrap/domain/Seed4JCliLauncher.java` recebeu `childMode` como estado de construção e expõe apenas `launch(String[] args)` para o fluxo principal.
- Os testes de `Seed4JCliLauncherFactoryTest`, `Seed4JCliLauncherTest` e `ExtensionRuntimeBootstrapInProcessTest` foram ajustados para cobrir o novo contrato.

#### Validation

- Command: `./mvnw -Dtest=Seed4JCliLauncherFactoryTest,Seed4JCliLauncherTest,ExtensionRuntimeBootstrapInProcessTest test`
- Result: testes verdes. O cenário `childMode=true` continua chamando o local runner sem ler o runtime mode.

#### Acceptance Criteria

- `Seed4JCliLauncherFactory.create(...)` recebe `PreSpringRuntimeEnvironment`.
- `LauncherDependencies` não contém `javaExecutable`.
- `Seed4JCliLauncher.launch(String[] args)` preserva standard mode, extension mode, fallback local e child mode.
- Não existe caller de `launcher.launch(args, childMode)` no source.

### Milestone 2 - Simplificar o application service e remover abstrações intermediárias

#### Goal

Fazer `PreSpringBootstrapApplicationService` parecer com um use case simples, no estilo de `RuntimeExtensionApplicationService`.

#### Changes

- `src/main/java/com/seed4j/cli/bootstrap/application/PreSpringBootstrapApplicationService.java` passou a receber `Seed4JCliLauncher` no construtor.
- `exitCodeFor(PreSpringBootstrapCommand command)` agora chama apenas `seed4jCliLauncher.launch(command.args())`.
- `src/main/java/com/seed4j/cli/bootstrap/application/PreSpringLauncher.java` e `PreSpringLauncherFactory.java` foram removidos.
- Os testes de `PreSpringBootstrapApplicationServiceTest`, `PreSpringBootstrapRunnerTest` e `Seed4JCliAppTest` foram ajustados para usar launcher real criado pela factory em vez das abstrações removidas.

#### Validation

- Command: `./mvnw -Dtest=PreSpringBootstrapApplicationServiceTest,PreSpringBootstrapRunnerTest,Seed4JCliAppTest test`
- Result: testes verdes. O application service delega apenas os `args` ao launcher.

#### Acceptance Criteria

- `PreSpringBootstrapApplicationService` possui um único campo de domínio: `Seed4JCliLauncher`.
- `PreSpringRuntimeEnvironmentReader` não é importado em `bootstrap/application/PreSpringBootstrapApplicationService.java`.
- `PreSpringLauncher` e `PreSpringLauncherFactory` não existem mais.
- O runner primário continua aceitando `String[] args` e criando `PreSpringBootstrapCommand`.

### Milestone 3 - Centralizar snapshot na composition

#### Goal

Fazer `PreSpringBootstrapConfiguration` ser o único ponto que lê o ambiente pré-Spring e monta adapters concretos com esse snapshot.

#### Changes

- `src/main/java/com/seed4j/cli/bootstrap/composition/PreSpringBootstrapConfiguration.java` passou a criar `CurrentProcessPreSpringRuntimeEnvironmentReader`, chamar `current()` uma vez e delegar para uma sobrecarga que recebe `PreSpringRuntimeEnvironment`.
- O helper privado `seed4jCliLauncher(PreSpringRuntimeEnvironment runtimeEnvironment)` monta `SpringBootLocalCliRunner`, `FileSystemRuntimeModeConfigurationRepository`, `JavaChildProcessCommandExecutor` e `Seed4JCliLauncherFactory`.
- `SpringBootLocalCliRunner` e `FileSystemRuntimeModeConfigurationRepository` continuam recebendo `runtimeEnvironment.userHomePath()` explicitamente.
- `src/test/java/com/seed4j/cli/bootstrap/composition/PreSpringBootstrapConfigurationTest.java` foi atualizado para injetar snapshot direto.

#### Validation

- Command: `./mvnw -Dtest=PreSpringBootstrapConfigurationTest,CurrentProcessPreSpringRuntimeEnvironmentReaderTest,SpringBootLocalCliRunnerTest,FileSystemRuntimeModeConfigurationRepositoryTest test`
- Result: testes verdes. A composition usa snapshot explícito e o reader secondary continua testado isoladamente.

#### Acceptance Criteria

- O ambiente técnico é lido uma única vez na composition de produção.
- `System.getProperty(...)` continua concentrado em `CurrentProcessPreSpringRuntimeEnvironmentReader`, exceto leituras já existentes em testes.
- A porta `PreSpringRuntimeEnvironmentReader` permanece no domínio.
- A composition monta o grafo, mas não executa o caso de uso.

### Milestone 4 - Regressão focada e documentação do resultado

#### Goal

Confirmar que o refactor não mudou comportamento público e registrar o que foi validado.

#### Changes

- Este ExecPlan foi atualizado com o resultado real da implementação.
- Não foi necessário alterar documentação de usuário, porque não havia texto vigente descrevendo `PreSpringLauncher` ou `PreSpringLauncherFactory`.
- A formatação foi corrigida com Prettier após a primeira checagem apontar um arquivo fora do padrão.

#### Validation

- Command: `./mvnw -Dtest=PreSpringBootstrapApplicationServiceTest,PreSpringBootstrapRunnerTest,PreSpringBootstrapConfigurationTest,Seed4JCliAppTest,Seed4JCliLauncherFactoryTest,Seed4JCliLauncherTest,ExtensionRuntimeBootstrapInProcessTest test`
- Result: todos os testes focados verdes.
- Command: `npm run prettier:check`
- Result: formatação verde.
- Gate final pendente: pedir ao usuário para executar `./mvnw clean verify` localmente e enviar o exit code mais um resumo de falhas, se houver.

#### Acceptance Criteria

- Testes focados passam.
- Prettier passa.
- O usuário confirma `./mvnw clean verify` verde ou fornece falha para investigação.
- O plano registra decisões e aprendizados finais.

## Progress

- [x] Baseline atual do repositório inspecionado em 2026-06-02.
- [x] Decisão arquitetural registrada: composition externa com snapshot de ambiente.
- [x] Decisão registrada: manter `PreSpringRuntimeEnvironmentReader` como porta.
- [x] Milestone 1 started.
- [x] Milestone 1 completed.
- [x] Milestone 2 started.
- [x] Milestone 2 completed.
- [x] Milestone 3 started.
- [x] Milestone 3 completed.
- [x] Milestone 4 started.
- [ ] Milestone 4 completed.
- [x] Testes focados executados com sucesso.
- [x] `npm run prettier:check` executado com sucesso.
- [ ] Aguardando o usuário executar `./mvnw clean verify` localmente e enviar o exit code e o resumo de falhas, se houver.

## Decisions

- Decision: Usar composição externa em vez de fazer `PreSpringBootstrapApplicationService` montar o launcher.
  Rationale: O application service deve ser apenas o caso de uso, enquanto a composition manual faz o papel que o Spring faria em runtime normal.
  Date/Author: 2026-06-02 / Codex e Renan

- Decision: Passar apenas `args` do application service para o launcher.
  Rationale: `childMode` é informação técnica já conhecida pela composition através de `PreSpringRuntimeEnvironment`; não deve atravessar o método do use case.
  Date/Author: 2026-06-02 / Codex e Renan

- Decision: Alterar `Seed4JCliLauncherFactory` para receber `PreSpringRuntimeEnvironment`.
  Rationale: O snapshot reúne os valores técnicos necessários para construir o launcher e evita parâmetros soltos de `Path` que podem ser combinados incorretamente.
  Date/Author: 2026-06-02 / Codex e Renan

- Decision: Manter `PreSpringRuntimeEnvironmentReader`.
  Rationale: Mesmo não sendo usado pelo application service, ele continua sendo uma porta do contexto bootstrap para acessar o ambiente técnico do processo via adapter secondary.
  Date/Author: 2026-06-02 / Codex e Renan

- Decision: Manter o construtor package-private antigo de `Seed4JCliLauncher` como overload delegando para o novo construtor com `childMode=false`.
  Rationale: Isso reduz churn em testes de scaffolding sem reintroduzir o método público `launch(args, childMode)` e sem afetar a composição de produção, que usa o construtor snapshot-aware.
  Date/Author: 2026-06-02 / Codex

## Risks and Mitigations

- Risk: Refactor quebrar child mode e causar recursão ou inicialização incorreta do CLI.
  Mitigation: Cobrir explicitamente o cenário `childMode=true` em `Seed4JCliLauncherTest` e `ExtensionRuntimeBootstrapInProcessTest`.

- Risk: `Seed4JCliLauncherTest` ficar repetitivo porque há muitas instanciações diretas do launcher.
  Mitigation: Preservar o construtor package-private anterior como overload delegando para o snapshot-aware.

- Risk: A composition virar canal de execução do caso de uso.
  Mitigation: `PreSpringBootstrapConfiguration` apenas cria objetos e retorna `PreSpringBootstrapRunner`; a execução continua em `PreSpringBootstrapRunner.exitCodeFor(...)`.

- Risk: Espalhar leitura de `System.getProperty(...)`.
  Mitigation: Manter essas leituras em `CurrentProcessPreSpringRuntimeEnvironmentReader` e validar com `rg`.

- Risk: Mudança parecer só movimento interno e esconder regressão pública.
  Mitigation: Rodar testes de launcher, app, primary runner e composition; pedir `./mvnw clean verify` ao usuário como gate final.

## Validation Strategy

1. Rodar testes focados de domínio do launcher após o Milestone 1.
2. Rodar testes focados de application, primary e app após o Milestone 2.
3. Rodar testes focados de composition e secondary após o Milestone 3.
4. Rodar o conjunto focado completo e `npm run prettier:check` após o Milestone 4.
5. Solicitar ao usuário o gate final `./mvnw clean verify`, conforme política do repositório.

## Rollout and Recovery

Este é um refactor interno sem alteração intencional de CLI pública. O rollout seguro é manter os commits focados e validar primeiro com testes específicos. Se houver regressão em startup, runtime mode ou extension mode, reverter o change set e restaurar temporariamente o contrato anterior do launcher até isolar o cenário quebrado.

## Lessons Learned

- `PreSpringBootstrapApplicationService` fica mais claro quando recebe o launcher pronto e não conhece factory nem reader.
- A composition é o ponto certo para ler o ambiente pré-Spring uma única vez e convertê-lo em snapshot.
- Os testes de application e runner ficam mais estáveis quando usam um launcher real criado pela factory, mesmo que o cenário seja controlado por `childMode=true`.
- Prettier só apontou um arquivo após a refatoração; a correção foi localizada.
