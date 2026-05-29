# Refatorar ambiente pré-Spring para porta secondary e composição manual

This ExecPlan is a living document. Keep `Progress`, `Decisions`, `Risks`, and `Lessons Learned` up to date as work advances.

Safety boundary: This task is limited to authorized, defensive maintenance of this repository. Do not provide offensive guidance or policy-bypassing instructions.

## Purpose / Big Picture

O objetivo é manter o comportamento observável do CLI igual, mas corrigir a fronteira arquitetural do bootstrap pré-Spring. `Seed4JCliApp` deve continuar sendo cliente fino do módulo `bootstrap`; `PreSpringBootstrapRunner` deve ser o adapter primário; leituras de ambiente do processo via `System.getProperty(...)` devem ficar em um adapter secondary acessado por uma porta de `application`; `bootstrap/composition` deve apenas montar objetos, como o Spring faria em runtime normal.

## Scope

In-scope:
- Mover leitura de `user.home`, `java.home`, `user.dir`, `sun.java.command`, `java.class.path` e `seed4j.cli.runtime.child` para um adapter secondary.
- Mover resolução de executable path e versão atual para o mesmo adapter de ambiente do processo.
- Consolidar `bootstrap/composition` em uma única classe de composição.
- Manter `PreSpringBootstrapRunner` como adapter primário.
- Atualizar testes do bootstrap pré-Spring.
- Basear a nomeação de classes, tipos e portas deste refactor no módulo `module` do projeto `seed4j`, consultando diretamente `/home/renanfranca/projects/seed4j/src/main/java/com/seed4j/module`.

Out-of-scope:
- Alterar regras de domínio em `Seed4JCliLauncher`.
- Refatorar comandos pós-Spring (`extension install`, `enable`, `disable`).
- Alterar mensagens, opções CLI ou contrato de `seed4j --version`.

## Definitions

- Pré-Spring: código executado antes de `SpringApplicationBuilder.run(...)`.
- Primary adapter: adapter que recebe a entrada externa e dirige o caso de uso. Aqui: `PreSpringBootstrapRunner`.
- Secondary adapter: adapter chamado pelo caso de uso para acessar infraestrutura externa. Aqui: ambiente do processo, filesystem, processo filho.
- Composition: pacote responsável apenas por instanciar e conectar objetos concretos.
- Referência de nomeação: o módulo `module` do projeto `seed4j` no caminho `/home/renanfranca/projects/seed4j/src/main/java/com/seed4j/module` é a fonte de verdade para nomes de novos tipos deste plano.

## Existing Context

Antes do refactor, `Seed4JCliApp` lia propriedades do processo e resolvia executable path antes de chamar o canal primário. Após Milestone 3 e alinhamento de nomenclatura, `Seed4JCliApp` delega ao `PreSpringBootstrapRunner`, que recebe `String[] args`, monta `PreSpringBootstrapCommand` e chama `PreSpringBootstrapApplicationService`; o wiring concreto fica centralizado em `PreSpringBootstrapConfiguration`.

## Desired End State

`Seed4JCliApp` chama apenas um resolver de exit code ligado ao primary adapter. `PreSpringBootstrapRunner` recebe só `String[] args` e cria um command de entrada. `PreSpringBootstrapApplicationService` consome uma porta `PreSpringRuntimeEnvironmentReader`, obtém o ambiente atual, cria o launcher e executa. `bootstrap/composition` terá uma única classe, `PreSpringBootstrapConfiguration`, que monta o grafo concreto sem ler `System.getProperty(...)`.

## Milestones

### Milestone 1 - Introduzir porta de ambiente pré-Spring

#### Goal

Modelar o ambiente do processo como dependência secondary do caso de uso, sem mudar comportamento.

#### Changes

- [x] Criar `src/main/java/com/seed4j/cli/bootstrap/application/PreSpringRuntimeEnvironment.java` como record com `userHomePath`, `executablePath`, `currentSeed4JVersion`, `childMode`, `javaExecutablePath`.
- [x] Criar `src/main/java/com/seed4j/cli/bootstrap/application/PreSpringRuntimeEnvironmentReader.java` com método `current()`.
- [x] Reduzir `PreSpringBootstrapCommand` para conter apenas `String[] args`.
- [x] Alterar `PreSpringLauncherFactory.create(...)` para receber também `javaExecutablePath`.

#### Validation

- [x] Command: `./mvnw -Dtest=PreSpringBootstrapApplicationServiceTest test`
- [x] Expected result: teste verde provando que o service obtém ambiente pela porta e delega `args`, `childMode`, paths e versão ao launcher.

#### Acceptance Criteria

- [x] Dados de ambiente não entram mais no command vindo do primary.
- [x] O service depende de uma porta explícita para ambiente do processo.

### Milestone 2 - Criar secondary adapter para o ambiente do processo

#### Goal

Remover de `Seed4JCliApp` a responsabilidade de ler propriedades do sistema e resolver executable path.

#### Changes

- [x] Criar `src/main/java/com/seed4j/cli/bootstrap/infrastructure/secondary/CurrentProcessPreSpringRuntimeEnvironmentReader.java`.
- [x] Mover para essa classe a leitura de system properties, resolução de `javaExecutablePath`, resolução de executable path e fallback `0.0.0-SNAPSHOT`.
- [x] Mover os testes de resolução de executable path de `Seed4JCliAppTest` para `CurrentProcessPreSpringRuntimeEnvironmentReaderTest`.
- [x] Garantir que `Seed4JCliApp` não chame `System.getProperty(...)`.

#### Validation

- [x] Command: `./mvnw -Dtest=CurrentProcessPreSpringRuntimeEnvironmentReaderTest,Seed4JCliAppTest test`
- [x] Expected result: testes verdes; resolução por code source, `sun.java.command` relativo/absoluto e classpath continua coberta.

#### Acceptance Criteria

- [x] Leituras `System.getProperty(...)` do bootstrap pré-Spring ficam no secondary adapter.
- [x] `Seed4JCliApp` deixa de conhecer `user.home`, `java.home`, `user.dir`, `sun.java.command`, `java.class.path` e `seed4j.cli.runtime.child`.

### Milestone 3 - Consolidar composition e preservar primary

#### Goal

Fazer `composition` apenas montar o grafo e manter `PreSpringBootstrapRunner` como canal primário.

#### Changes

- [x] Criar `src/main/java/com/seed4j/cli/bootstrap/composition/PreSpringBootstrapConfiguration.java`.
- [x] Mover para essa classe a montagem de `PreSpringBootstrapRunner`, `PreSpringBootstrapApplicationService`, `CurrentProcessPreSpringRuntimeEnvironmentReader`, factory de launcher, adapters Spring, `FileSystemRuntimeModeConfigurationRepository` e `JavaChildProcessCommandExecutor`.
- [x] Remover `src/main/java/com/seed4j/cli/bootstrap/composition/InfrastructurePreSpringLauncherFactory.java`.
- [x] Alterar `PreSpringBootstrapRunner` para receber `PreSpringBootstrapApplicationService` no construtor e expor `exitCodeFor(String[] args)`.
- [x] Alterar `Seed4JCliApp` para obter o primary pela composition e chamar o primary, não a composition como canal de caso de uso.

#### Validation

- [x] Command: `./mvnw -Dtest=PreSpringBootstrapRunnerTest,PreSpringBootstrapConfigurationTest,Seed4JCliAppTest test`
- [x] Expected result: testes verdes provando `Seed4JCliApp -> primary -> application` e composition apenas como wiring.
- [x] Command: `rg -n "System\\.getProperty|InfrastructurePreSpringLauncherFactory" src/main/java/com/seed4j/cli/Seed4JCliApp.java src/main/java/com/seed4j/cli/bootstrap/composition src/main/java/com/seed4j/cli/bootstrap/infrastructure/primary`
- [x] Expected result: nenhum match para `System.getProperty` fora do secondary; nenhum match para a classe removida.

#### Acceptance Criteria

- [x] `bootstrap/composition` tem uma única classe pública de composição.
- [x] `PreSpringBootstrapRunner` continua existindo como adapter primário.
- [x] Composition não executa o caso de uso; só monta objetos.

### Milestone 4 - Regressão completa do CLI

#### Goal

Confirmar que o refactor não alterou comportamento observável.

#### Changes

- [ ] Atualizar imports, testes e nomes quebrados pela remoção da factory antiga.
- [ ] Rodar formatação apenas se `prettier:check` indicar divergência.

#### Validation

- [ ] Command: `./mvnw -Dtest=Seed4JCliAppTest,PreSpringBootstrapApplicationServiceTest,PreSpringBootstrapRunnerTest,CurrentProcessPreSpringRuntimeEnvironmentReaderTest,PreSpringBootstrapConfigurationTest,Seed4JCliLauncherFactoryTest,Seed4JCliLauncherTest test`
- [ ] Expected result: testes focados verdes.
- [ ] Command: `./mvnw clean verify`
- [ ] Expected result: validação completa verde.
- [ ] Command: `npm run prettier:check`
- [ ] Expected result: sem divergências.
- [ ] Command: `seed4j --version`
- [ ] Expected result: comando executa com exit code `0` e saída de versão compatível.

#### Acceptance Criteria

- [ ] `seed4j --version` mantém comportamento.
- [ ] Full verification passa sem regressão.

## Progress

- [x] ExecPlan drafted
- [x] Milestone 1 started
- [x] Milestone 1 completed
- [x] Milestone 2 started
- [x] Milestone 2 completed
- [x] Milestone 3 started
- [x] Milestone 3 completed
- [ ] Milestone 4 started
- [ ] Milestone 4 completed

## Decisions

- Decision: `System.getProperty(...)` do bootstrap pré-Spring será lido por secondary adapter.
  Rationale: ambiente do processo é infraestrutura externa consultada pelo caso de uso, não responsabilidade de composition nem do cliente `Seed4JCliApp`.
  Date/Author: 2026-05-29 / Renan + Codex

- Decision: `PreSpringBootstrapApplicationService` consumirá `PreSpringRuntimeEnvironmentReader`.
  Rationale: preserva o fluxo hexagonal: primary dirige application; application chama portas; secondary implementa infraestrutura.
  Date/Author: 2026-05-29 / Renan + Codex

- Decision: `bootstrap/composition` terá uma única classe `PreSpringBootstrapConfiguration`.
  Rationale: composition deve fazer apenas wiring manual, substituindo o papel que Spring Boot teria no pré-Spring.
  Date/Author: 2026-05-29 / Renan + Codex

- Decision: `PreSpringBootstrapRunner` será mantido como adapter primário.
  Rationale: `Seed4JCliApp` é cliente do módulo bootstrap e não deve falar diretamente com composition para executar o caso de uso.
  Date/Author: 2026-05-29 / Renan + Codex

- Decision: durante o Milestone 1, o primary cria um `PreSpringRuntimeEnvironmentReader` ad-hoc por chamada para manter comportamento atual até o adapter secondary dedicado do Milestone 2.
  Rationale: preserva o loop TDD do milestone e evita antecipar responsabilidade de leitura de ambiente antes do secondary adapter existir.
  Date/Author: 2026-05-29 / Renan + Codex

- Decision: no Milestone 2, `PreSpringBootstrapRunner` passou a receber `PreSpringRuntimeEnvironment` explícito e deixou de ler `java.home` diretamente.
  Rationale: elimina leitura de `System.getProperty(...)` fora do secondary adapter sem antecipar a remoção do primary planejada para o Milestone 3.
  Date/Author: 2026-05-29 / Renan + Codex

- Decision: a nomeação de novos tipos deste refactor deve seguir o padrão adotado no módulo `module` de `seed4j`, com consulta direta ao caminho `/home/renanfranca/projects/seed4j/src/main/java/com/seed4j/module` antes de fechar nomes.
  Rationale: reduz deriva de vocabulário entre projetos e mantém consistência arquitetural entre `seed4j-cli` e `seed4j`.
  Date/Author: 2026-05-29 / Renan + Codex

- Decision: no Milestone 3, a montagem concreta do launcher pré-Spring foi incorporada em `PreSpringBootstrapConfiguration` e o adapter primário foi reduzido para delegação ao application service.
  Rationale: elimina a factory de composição transitória, preserva o canal primário explícito e mantém a composition restrita a wiring.
  Date/Author: 2026-05-29 / Renan + Codex

## Risks and Mitigations

- Risk: confundir composition com canal de execução.
  Mitigation: `PreSpringBootstrapConfiguration` expõe apenas método de montagem do primary, não `exitCodeFor(...)`.

- Risk: perda de cobertura ao mover resolução de executable path.
  Mitigation: migrar testes existentes de `Seed4JCliAppTest` para o teste do secondary adapter no mesmo milestone.

- Risk: regressão em modo extension por mudança em `javaExecutablePath` ou `childMode`.
  Mitigation: manter semântica atual e rodar testes de launcher além dos testes de bootstrap.

- Risk: dependency cycle conceitual com `Seed4JCliApp.class` usado no Spring builder e code source.
  Mitigation: manter essa referência restrita à composition e ao secondary adapter de ambiente, nunca ao application.

## Validation Strategy

1. Rodar testes focados por milestone.
2. Rodar suíte focada de bootstrap e launcher.
3. Rodar `./mvnw clean verify`.
4. Rodar `npm run prettier:check`.
5. Exercitar manualmente `seed4j --version`.

## Rollout and Recovery

Rollout:
1. Implementar em milestones pequenos e manter o plano atualizado.
2. Validar o fluxo focado antes da verificação completa.
3. Publicar sem mudança de contrato funcional do CLI.

Recovery:
1. Se `Seed4JCliApp` falhar, reverter Milestone 3.
2. Se resolução de paths falhar, reverter Milestone 2.
3. Se contratos de application ficarem instáveis, reverter Milestone 1.
4. Após qualquer rollback parcial, rodar testes focados e `./mvnw clean verify`.

## Lessons Learned

- `composition` no pré-Spring deve substituir apenas o wiring que Spring faria, não virar adapter primário.
- Leituras de ambiente do processo são infraestrutura externa quando o application precisa delas para orquestrar o caso de uso.
- Manter o primary explícito melhora a leitura da fronteira entre `Seed4JCliApp` e o módulo `bootstrap`.
- O passo intermediário com provider ad-hoc no primary permite migrar o contract da application sem quebrar o comportamento observável antes do Milestone 2.
- Em execuções focadas de testes, `jacoco:report` pode falhar transitoriamente com `EOFException`; repetir o mesmo comando confirmou estabilidade antes de tratar como regressão real.
- Ao reduzir o primary para `exitCodeFor(String[] args)`, os testes de integração de runtime devem migrar para `composition`; manter esse teste em `Seed4JCliAppTest` força acoplamento indevido com detalhes de wiring.
