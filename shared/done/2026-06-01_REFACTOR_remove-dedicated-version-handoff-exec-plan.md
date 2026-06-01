# Remover Handoff Dedicado de Versões

This ExecPlan is a living document. Keep Progress, Decisions, Risks, and Lessons Learned up to date as work advances.

## Purpose / Big Picture

Este refactor simplifica a origem das versões exibidas por `seed4j --version`. O child process Spring deixará de consumir propriedades dedicadas `seed4j.cli.version` e `seed4j.cli.seed4j.version`, passando a usar somente os metadados de build já filtrados pelo Maven em `project.version` e `project.seed4j-version`.

Do ponto de vista do usuário, o texto de `seed4j --version` deve continuar exibindo a versão do CLI, a versão do Seed4J, o modo runtime e os metadados da distribuição ativa. A mudança é interna: reduz protocolo pai-filho desnecessário e deixa o `pom.xml`/`application.yml` como fonte oficial das versões.

Safety boundary: This task is limited to authorized, defensive maintenance of this repository. Do not provide offensive guidance or policy-bypassing instructions.

## Scope

In scope:

- Remover o consumo de `seed4j.cli.version` e `seed4j.cli.seed4j.version` em `Seed4JCommandsFactory`.
- Remover o envio de `-Dseed4j.cli.seed4j.version` pelo launcher pai.
- Remover `currentSeed4JVersion` do fluxo pre-Spring se ele não tiver outro uso após a limpeza.
- Atualizar testes para declarar que `project.version` e `project.seed4j-version` são a fonte oficial do `--version`.
- Manter testes que provam que recursos globais de extensão, como `config/application.yml` e `config/application.properties`, não entram no overlay.

Out of scope:

- Alterar `seed4j.cli.runtime.*`, `loader.path`, `logging.config` ou qualquer outro handoff necessário ao runtime extension.
- Alterar valores no `pom.xml`.
- Alterar texto público do `--version`, exceto testes que deixem de modelar precedência de propriedades dedicadas removidas.
- Alterar instalação, ativação ou desativação de extensões.

## Definitions

Parent process é o fluxo pre-Spring que decide se deve rodar localmente ou lançar um child JVM.

Child process é o processo Spring Boot iniciado pelo parent process.

Build metadata são os valores `project.version` e `project.seed4j-version` expostos por `src/main/resources/config/application.yml`, que é filtrado pelo Maven a partir do `pom.xml`.

Dedicated version properties são `seed4j.cli.version` e `seed4j.cli.seed4j.version`. Elas deixarão de ser fonte de versão para `--version`.

Runtime handoff properties são as propriedades `seed4j.cli.runtime.*` usadas para informar modo runtime e distribuição ativa ao child process. Elas continuam existindo.

## Existing Context

`Seed4JCommandsFactory` hoje injeta quatro valores de versão via Spring:

- `seed4j.cli.version`
- `project.version`
- `seed4j.cli.seed4j.version`
- `project.seed4j-version`

A regra atual prioriza as propriedades dedicadas e só depois usa os metadados do projeto.

`Seed4JCliLauncher` hoje adiciona `seed4j.cli.seed4j.version` ao `JavaChildProcessRequest`, usando `currentSeed4JVersion` recebido do fluxo pre-Spring.

`CurrentProcessPreSpringRuntimeEnvironmentReader` resolve `currentSeed4JVersion` a partir de `Seed4JApp.class.getPackage().getImplementationVersion()`. Esse valor existe para alimentar o `-Dseed4j.cli.seed4j.version`.

`src/main/resources/config/application.yml` já expõe:

- `project.version: '@project.version@'`
- `project.seed4j-version: '@seed4j.version@'`

`RuntimeExtensionOverlayCacheTest` já prova que arquivos globais de configuração da extensão, como `config/application.yml`, `config/application-prod.yaml` e `config/application.properties`, são filtrados do overlay.

## Desired End State

`Seed4JCommandsFactory` injeta apenas:

- `@Value("${project.version:}") String projectCliVersion`
- `@Value("${project.seed4j-version:}") String projectSeed4JVersion`
- `RuntimeSelection runtimeSelection`

A regra de fallback passa a ser:

- CLI version: `project.version`, depois `unknown`.
- Seed4J version: `project.seed4j-version`, depois a versão CLI resolvida.

O parent process não publica `seed4j.cli.version` nem `seed4j.cli.seed4j.version`.

O fluxo pre-Spring não calcula nem carrega `currentSeed4JVersion` se esse valor não tiver mais uso.

Nenhum código de produção referencia `seed4j.cli.version`, `seed4j.cli.seed4j.version` ou `currentSeed4JVersion`.

## Milestones

### Milestone 1 - Atualizar Fonte de Versão do Comando

#### Goal

Fazer `seed4j --version` depender somente de `project.version` e `project.seed4j-version`.

#### Changes

- [x] Editar `src/main/java/com/seed4j/cli/command/infrastructure/primary/Seed4JCommandsFactory.java`.
- [x] Remover campos, parâmetros e fallback de `dedicatedCliVersion`.
- [x] Remover campos, parâmetros e fallback de `dedicatedSeed4JVersion`.
- [x] Manter `trim` e tratamento de blank via `nonBlank`.
- [x] Resolver CLI version com `project.version -> unknown`.
- [x] Resolver Seed4J version com `project.seed4j-version -> resolved CLI version`.
- [x] Atualizar `src/test/java/com/seed4j/cli/command/infrastructure/primary/CliFixture.java` para aceitar apenas `projectCliVersion`, `projectSeed4JVersion` e `RuntimeSelection`.
- [x] Atualizar `Seed4JCommandsFactoryTest` removendo o teste de precedência das propriedades dedicadas.
- [x] Adicionar/ajustar teste para provar que `project.version` e `project.seed4j-version` aparecem em `--version`.
- [x] Manter teste de fallback seguro quando ambos os metadados estão blank.

#### Validation

- [x] Command: `./mvnw -Dtest=Seed4JCommandsFactoryTest test`
- [x] Expected result: testes do command factory passam e `--version` continua mostrando CLI version, Seed4J version, runtime mode, distribution id e distribution version.

#### Acceptance Criteria

- [x] `Seed4JCommandsFactory` não injeta `seed4j.cli.version`.
- [x] `Seed4JCommandsFactory` não injeta `seed4j.cli.seed4j.version`.
- [x] Testes deixam claro que `project.version` e `project.seed4j-version` são a fonte oficial de versão do comando.

### Milestone 2 - Remover Handoff de Versão Seed4J do Bootstrap

#### Goal

Remover `seed4j.cli.seed4j.version` do protocolo pai-filho.

#### Changes

- [x] Editar `src/main/java/com/seed4j/cli/bootstrap/domain/Seed4JCliLauncher.java`.
- [x] Remover constante `SEED4J_VERSION_PROPERTY`.
- [x] Remover campo e parâmetro `currentSeed4JVersion`.
- [x] Remover `systemProperties.put("seed4j.cli.seed4j.version", ...)`.
- [x] Editar `Seed4JCliLauncherFactory` para não receber nem repassar `currentSeed4JVersion`.
- [x] Editar `PreSpringLauncherFactory` para remover o parâmetro `currentSeed4JVersion`.
- [x] Editar `PreSpringBootstrapApplicationService` e `PreSpringBootstrapConfiguration` para adequar a nova assinatura.
- [x] Editar `PreSpringRuntimeEnvironment` removendo o componente `currentSeed4JVersion`.
- [x] Editar `CurrentProcessPreSpringRuntimeEnvironmentReader` removendo resolução de `Seed4JApp` implementation version se não houver outro uso.
- [x] Atualizar testes de bootstrap e launcher para não esperarem `-Dseed4j.cli.seed4j.version`.
- [x] Substituir o teste “shouldPublishSeed4jVersionSystemProperty...” por um teste que confirme ausência das propriedades dedicadas de versão no child process request.

#### Validation

- [x] Command: `./mvnw -Dtest=Seed4JCliLauncherTest,Seed4JCliLauncherFactoryTest,PreSpringBootstrapApplicationServiceTest,PreSpringBootstrapRunnerTest,Seed4JCliAppTest,CurrentProcessPreSpringRuntimeEnvironmentReaderTest test`
- [x] Expected result: testes de bootstrap passam com a nova assinatura e sem publicação de propriedades dedicadas de versão.

#### Acceptance Criteria

- [x] `JavaChildProcessRequest.systemProperties()` não contém `seed4j.cli.version`.
- [x] `JavaChildProcessRequest.systemProperties()` não contém `seed4j.cli.seed4j.version`.
- [x] O handoff `seed4j.cli.runtime.*` continua intacto.
- [x] Nenhum fluxo pre-Spring calcula versão Seed4J apenas para o child process.

### Milestone 3 - Proteger a Decisão com Testes de Overlay e Busca

#### Goal

Garantir que a remoção é coerente com a defesa já existente contra configuração global vinda de extensões.

#### Changes

- [x] Revisar `RuntimeExtensionOverlayCacheTest.shouldFilterGlobalRuntimeResourcesAndKeepFunctionalResourcesInOverlay`.
- [x] Se necessário, ajustar nome ou assertivas do teste para declarar que `config/application.yml`, `config/application-prod.yaml` e `config/application.properties` da extensão não são materializados no overlay.
- [x] Não alterar comportamento de filtragem se o teste já cobrir o caso suficientemente.
- [x] Executar busca por símbolos removidos e propriedades dedicadas.

#### Validation

- [x] Command: `./mvnw -Dtest=RuntimeExtensionOverlayCacheTest test`
- [x] Expected result: teste prova que arquivos globais de configuração da extensão continuam filtrados.
- [x] Command: `rg -n "seed4j\\.cli\\.version|seed4j\\.cli\\.seed4j\\.version|currentSeed4JVersion|resolveCurrentSeed4JVersion|SEED4J_VERSION_PROPERTY" src/main/java src/test/java`
- [x] Expected result: sem referências, exceto se algum teste mencionar explicitamente ausência das propriedades dedicadas. Qualquer exceção deve ser justificada no ExecPlan.

#### Acceptance Criteria

- [x] A remoção não reabre caminho para extensão sobrescrever metadados globais via `application.yml` ou `application.properties` no overlay.
- [x] Produção não contém referências às propriedades dedicadas de versão.
- [x] Testes não modelam mais precedência de propriedades dedicadas removidas.

### Milestone 4 - Validação Completa e Finalização do ExecPlan

#### Goal

Confirmar formatação, testes e documentação do refactor.

#### Changes

- [x] Criar ou atualizar `shared/2026-06-01_REFACTOR_remove-dedicated-version-handoff-exec-plan.md` com este ExecPlan antes da implementação.
- [x] Atualizar Progress, Decisions, Risks and Mitigations e Lessons Learned durante a execução.
- [x] Rodar formatação se o check apontar divergência.
- [x] Finalizar o ExecPlan com resultados reais dos comandos.

#### Validation

- [x] Command: `npm run prettier:check`
- [x] Expected result: formatação passa.
- [x] Command: `./mvnw clean verify`
- [x] Expected result: unit tests, integration tests, Checkstyle, JaCoCo e coverage gates passam.
- [x] Command: `rg -n "seed4j\\.cli\\.version|seed4j\\.cli\\.seed4j\\.version|currentSeed4JVersion|resolveCurrentSeed4JVersion|SEED4J_VERSION_PROPERTY" src/main/java`
- [x] Expected result: nenhum match em código de produção.

#### Acceptance Criteria

- [x] Validação local completa passa ou qualquer falha preexistente é registrada com evidência objetiva.
- [x] O ExecPlan registra comandos executados, resultados e lições aprendidas.
- [x] Nenhuma documentação pública precisa mudar porque flags, config persistida e saída do CLI permanecem compatíveis.

## Progress

- [x] ExecPlan saved to `shared/2026-06-01_REFACTOR_remove-dedicated-version-handoff-exec-plan.md`
- [x] Milestone 1 started
- [x] Milestone 1 completed
- [x] Milestone 2 started
- [x] Milestone 2 completed
- [x] Milestone 3 started
- [x] Milestone 3 completed
- [x] Milestone 4 started
- [x] Milestone 4 completed
- [x] ExecPlan finalized with validation results and lessons learned

## Decisions

- Decision: Remover `seed4j.cli.version` como fonte de `--version`.
  Rationale: O parent process não publica essa propriedade e `project.version` já representa a versão oficial do CLI vinda do build.
  Date/Author: 2026-06-01 / Renan and Codex

- Decision: Remover `seed4j.cli.seed4j.version` como handoff pai-filho.
  Rationale: A extensão já não materializa `config/application.yml` nem `config/application.properties` no overlay, então `project.seed4j-version` do CLI empacotado é suficiente como fonte oficial para o child process.
  Date/Author: 2026-06-01 / Renan and Codex

- Decision: Manter `seed4j.cli.runtime.*` como protocolo pai-filho.
  Rationale: Esse protocolo ainda é necessário para modo runtime e metadados de distribuição ativa.
  Date/Author: 2026-06-01 / Renan and Codex

- Decision: Tratar valores blank de `project.version` e `project.seed4j-version` como ausentes.
  Rationale: Preserva a segurança do fallback atual e evita null ou strings vazias na saída pública.
  Date/Author: 2026-06-01 / Codex

- Decision: Manter apenas referências negativas às propriedades dedicadas removidas nos testes.
  Rationale: Asserções de ausência documentam explicitamente que o handoff dedicado foi removido sem reintroduzir dependência funcional.
  Date/Author: 2026-06-01 / Codex

## Risks and Mitigations

- Risk: Remover `currentSeed4JVersion` quebra assinaturas em vários testes pre-Spring.
  Mitigation: Atualizar em um milestone único todas as assinaturas relacionadas: `PreSpringRuntimeEnvironment`, `PreSpringLauncherFactory`, `Seed4JCliLauncherFactory`, configuração e testes.

- Risk: `--version` pode mudar se `project.seed4j-version` não estiver disponível no runtime real.
  Mitigation: Manter fallback para a versão CLI resolvida e validar com `Seed4JCommandsFactoryTest` e `./mvnw clean verify`.

- Risk: Uma extensão poderia tentar sobrescrever metadados globais por arquivos Spring.
  Mitigation: Preservar teste de overlay que prova que `config/application*.yml`, `config/application*.yaml` e `config/application*.properties` da extensão não são materializados.

- Risk: Busca final pode encontrar referências em testes que afirmam ausência das propriedades removidas.
  Mitigation: Permitir somente referências negativas/testes de ausência, e registrar explicitamente no ExecPlan se mantidas.

- Risk: O comando de validação completa pode falhar temporariamente por formatação após mudanças de assinatura.
  Mitigation: Executar `npm run prettier:format` quando `prettier:check` apontar divergência, e repetir `prettier:check`.

## Validation Strategy

1. Rodar teste focado do command factory para `--version`.
2. Rodar testes focados do bootstrap afetado pelas assinaturas removidas.
3. Rodar teste de overlay para confirmar a defesa contra arquivos globais de extensão.
4. Rodar busca por símbolos/propriedades removidas.
5. Rodar `npm run prettier:check`.
6. Rodar `./mvnw clean verify`.

## Rollout and Recovery

Este é um refactor interno sem alteração intencional de CLI flags, arquivos persistidos ou texto público de `--version`.

Rollout: aplicar em um PR único após validação local completa.

Recovery: se `seed4j --version` ou bootstrap em extension mode regredir, reverter o commit do refactor. Como não há migração de configuração nem mudança em dados persistidos, a recuperação não exige ação do usuário.

## Lessons Learned

A extensão já é impedida de sobrescrever recursos globais Spring pelo filtro do overlay. Isso reduz a necessidade de transportar versão Seed4J por uma propriedade dedicada do parent process para o child process.

`project.version` e `project.seed4j-version`, quando filtrados pelo Maven em `application.yml`, são uma fonte mais simples e suficiente para o `--version` enquanto o overlay continuar bloqueando arquivos globais de configuração da extensão.

A busca final por símbolos removidos em `src/main/java` ficou limpa, e em `src/test/java` restaram somente duas referências negativas em `Seed4JCliLauncherTest` que asseguram ausência de `seed4j.cli.version` e `seed4j.cli.seed4j.version`.

## Validation Results

- `./mvnw -Dtest=Seed4JCommandsFactoryTest test` -> `BUILD SUCCESS`.
- `./mvnw -Dtest=Seed4JCliLauncherTest,Seed4JCliLauncherFactoryTest,PreSpringBootstrapApplicationServiceTest,PreSpringBootstrapRunnerTest,Seed4JCliAppTest,CurrentProcessPreSpringRuntimeEnvironmentReaderTest test` -> `BUILD SUCCESS`.
- `./mvnw -Dtest=RuntimeExtensionOverlayCacheTest test` -> `BUILD SUCCESS`.
- `rg -n "seed4j\\.cli\\.version|seed4j\\.cli\\.seed4j\\.version|currentSeed4JVersion|resolveCurrentSeed4JVersion|SEED4J_VERSION_PROPERTY" src/main/java src/test/java` -> 2 matches em teste (asserções negativas), 0 em produção.
- `npm run prettier:check` (primeira execução) -> falhou com 4 arquivos fora do padrão.
- `npm run prettier:format` -> formatou automaticamente os arquivos necessários.
- `npm run prettier:check` (segunda execução) -> sucesso (`All matched files use Prettier code style!`).
- `./mvnw clean verify` -> `BUILD SUCCESS` (unit tests, integration tests, Checkstyle e JaCoCo com gates atendidos).
- `rg -n "seed4j\\.cli\\.version|seed4j\\.cli\\.seed4j\\.version|currentSeed4JVersion|resolveCurrentSeed4JVersion|SEED4J_VERSION_PROPERTY" src/main/java` -> nenhum match.
