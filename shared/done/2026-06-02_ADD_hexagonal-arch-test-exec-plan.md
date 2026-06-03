# Implementar HexagonalArchTest no seed4j-cli

This ExecPlan is a living document. Keep `Progress`, `Decisions`, `Risks and Mitigations`, and `Lessons Learned` up to date as work advances.

Safety boundary: This task is limited to authorized, defensive maintenance of this repository. Do not provide offensive guidance or policy-bypassing instructions.

## Purpose / Big Picture

Implementar um teste automatizado de arquitetura hexagonal para o `seed4j-cli`, inspirado em `/home/renanfranca/projects/seed4j/src/test/java/com/seed4j/HexagonalArchTest.java`. O comportamento observável será: `./mvnw -Dtest=HexagonalArchTest test` falha quando uma camada viola os limites definidos, por exemplo `application` dependendo de `infrastructure` ou `domain` dependendo de Spring. Isso reduz regressões arquiteturais enquanto o bootstrap runtime, comandos CLI e shared kernels evoluem.

## Scope

In scope: adicionar ArchUnit como dependência de teste, criar `src/test/java/com/seed4j/cli/HexagonalArchTest.java`, adaptar regras do `seed4j` para os bounded contexts do CLI, e documentar explicitamente a exceção da composition root pre-Spring.

Out of scope: refatorar comportamento da CLI, mover `PreSpringBootstrapConfiguration`, alterar comandos Picocli, alterar runtime extension flow, ou portar todos os detalhes do `HexagonalArchTest` do `seed4j` sem adaptação.

Out of scope: executar automaticamente `./mvnw clean verify`; conforme as regras do repositório, pedir ao usuário para rodar esse gate final localmente quando necessário.

## Definitions

`Bounded Context`: pacote raiz anotado com `@BusinessContext`, atualmente `com.seed4j.cli.bootstrap` e `com.seed4j.cli.command`.

`Shared Kernel`: pacote raiz anotado com `@SharedKernel`, atualmente `com.seed4j.cli.shared.collection`, `com.seed4j.cli.shared.error` e `com.seed4j.cli.shared.generation`.

`Domain`: código em `..domain..`. Deve conter regras, tipos de negócio, portas e serviços de domínio puros.

`Application`: código em `..application..`. Deve orquestrar casos de uso e pode depender do domínio, mas não da infraestrutura.

`Primary adapter`: código em `..infrastructure.primary..`. Recebe chamadas externas, como comandos CLI ou runners, e chama application/domain conforme necessário.

`Secondary adapter`: código em `..infrastructure.secondary..`. Implementa portas e integra com filesystem, Spring Boot, processos ou outras tecnologias.

`Composition root`: código que monta grafo de objetos. No CLI existe `src/main/java/com/seed4j/cli/bootstrap/composition/PreSpringBootstrapConfiguration.java`, necessário porque parte do bootstrap roda antes do contexto Spring existir.

`ArchUnit`: biblioteca de teste que valida dependências entre pacotes/classes no bytecode Java.

## Existing Context

O `seed4j-cli` já possui `@BusinessContext` e `@SharedKernel` em `src/main/java/com/seed4j/cli/**/package-info.java`, mas não possui `HexagonalArchTest`.

O `pom.xml` do `seed4j-cli` não possui dependência ArchUnit. O projeto `seed4j` usa `com.tngtech.archunit:archunit-junit5-api` na versão `1.4.2`.

O `seed4j` valida regras como `application` não depender de `infrastructure`, `domain` depender apenas de domínios/shared kernels/libs autorizadas, `primary` não depender de `secondary`, e `secondary` não depender de `application`.

O `seed4j-cli` tem uma exceção real: `bootstrap/composition/PreSpringBootstrapConfiguration.java` monta manualmente application, primary, secondary e domain para o fluxo pre-Spring. Essa exceção precisa ser explicitamente modelada no teste para não virar uma brecha arquitetural genérica.

O `seed4j-cli` já tem `src/main/java/com/seed4j/cli/bootstrap/domain/Seed4JCliHome.java`, um tipo de domínio que encapsula caminhos derivados de `user.home`. Isso reforça que application pode depender de domínio, mas não deve ler ambiente global diretamente.

## Desired End State

`src/test/java/com/seed4j/cli/HexagonalArchTest.java` existe, usa `@UnitTest`, importa classes de produção com `ClassFileImporter`, descobre bounded contexts/shared kernels por `package-info.java`, e valida os limites arquiteturais principais do CLI.

O `pom.xml` possui propriedade `archunit-junit5.version` com valor `1.4.2` e dependência test-scope `archunit-junit5-api`.

O teste permite a composition root pre-Spring como exceção explícita e limitada: `PreSpringBootstrapConfiguration` pode montar o grafo antes do Spring, mas production code comum não deve depender de `..composition..`.

O teste roda com `./mvnw -Dtest=HexagonalArchTest test` e depois com `./mvnw test`.

## Milestones

### Milestone 1 - Adicionar ArchUnit e criar o teste base

#### Goal

Preparar o projeto para executar regras de arquitetura como teste unitário Maven.

#### Changes

- [ ] Editar `pom.xml` para adicionar `<archunit-junit5.version>1.4.2</archunit-junit5.version>` em `<properties>`.
- [ ] Editar `pom.xml` para adicionar dependência test-scope `com.tngtech.archunit:archunit-junit5-api`.
- [ ] Criar `src/test/java/com/seed4j/cli/HexagonalArchTest.java`.
- [ ] Anotar a classe com `@UnitTest`.
- [ ] Definir `ROOT_PACKAGE = "com.seed4j.cli"`.
- [ ] Importar classes de produção com `ClassFileImporter` e `ImportOption.Predefined.DO_NOT_INCLUDE_TESTS`.
- [ ] Copiar/adaptar helpers de descoberta de pacotes anotados com `@BusinessContext` e `@SharedKernel` a partir do `seed4j`.

#### Validation

- [ ] Command: `./mvnw -Dtest=HexagonalArchTest test`
- [ ] Expected result: o projeto compila e o teste executa, mesmo que falhe por regras ainda incompletas ou por violações reais.

#### Acceptance Criteria

- [ ] Maven resolve ArchUnit sem erro.
- [ ] `HexagonalArchTest` é descoberto pelo Surefire.
- [ ] O teste consegue importar classes de `com.seed4j.cli`.

### Milestone 2 - Implementar regras centrais de camadas

#### Goal

Proteger as fronteiras hexagonais principais sem tratar ainda a exceção de composition root.

#### Changes

- [ ] Adicionar regra `Domain.shouldNotDependOnOutside`.
- [ ] Autorizar domínio a depender apenas de `..domain..`, Java, pacote vazio ArchUnit, `org.slf4j..`, `org.apache.commons..`, `org.jspecify.annotations..`, `com.google.guava..` se necessário, e shared kernels.
- [ ] Adicionar regra `Application.shouldNotDependOnInfrastructure`.
- [ ] Adicionar regra `Primary.shouldNotDependOnSecondary`.
- [ ] Adicionar regra `Secondary.shouldNotDependOnApplication`.
- [ ] Adicionar regra `Secondary.shouldNotDependOnSameContextPrimary`.
- [ ] Adicionar regra `BoundedContexts.shouldNotDependOnOtherBoundedContextDomains`.
- [ ] Adicionar regra `SharedKernels.sharedPackageShouldOnlyContainSharedKernels`.
- [ ] Não adicionar regra proibindo `application -> domain`, porque isso contradiz o flavor usado pelo `seed4j` e pelos exemplos do sample app.

#### Validation

- [ ] Command: `./mvnw -Dtest=HexagonalArchTest test`
- [ ] Expected result: regras passam ou apontam violações arquiteturais concretas e pequenas.

#### Acceptance Criteria

- [ ] `RuntimeExtensionApplicationService` pode depender de `RuntimeExtensionConfiguration`, `RuntimeModeConfigurationRepository` e `RuntimeExtensionArtifactsRepository`.
- [ ] Nenhuma classe em `..application..` depende de `..infrastructure..`.
- [ ] Nenhuma classe em `..domain..` depende de Spring ou adapters.
- [ ] Violação real não deve ser silenciada sem registro em `Decisions`.

### Milestone 3 - Modelar a exceção da composition root pre-Spring

#### Goal

Permitir a composição manual necessária antes do Spring sem abrir uma brecha genérica na arquitetura.

#### Changes

- [ ] Adicionar uma seção `CompositionRoot` no `HexagonalArchTest`.
- [ ] Definir `COMPOSITION_PACKAGES = ROOT_PACKAGE.concat(".bootstrap.composition..")`.
- [ ] Adicionar regra: classes em `..composition..` não devem ser acessadas por bounded contexts, shared kernels, primary adapters, secondary adapters ou application services.
- [ ] Permitir acesso de `com.seed4j.cli.Seed4JCliApp` a `PreSpringBootstrapConfiguration`, porque esse é o ponto de entrada de produção.
- [ ] Não forçar `..composition..` a se comportar como `application`, `primary` ou `secondary`, porque sua responsabilidade é montar o grafo pre-Spring.
- [ ] Se for usada `Architectures.layeredArchitecture()`, adaptar a regra para que `composition` seja camada explícita permitida a acessar application, primary e secondary dentro do bootstrap.
- [ ] Se a regra de layered architecture ficar frágil por causa de `composition`, preferir regras ArchUnit diretas e explícitas, como no Milestone 2.

#### Validation

- [ ] Command: `./mvnw -Dtest=HexagonalArchTest test`
- [ ] Expected result: a exceção de `bootstrap.composition` é aceita somente para o caso pre-Spring previsto.

#### Acceptance Criteria

- [ ] `Seed4JCliApp` pode chamar `PreSpringBootstrapConfiguration`.
- [ ] Classes comuns de `bootstrap.application`, `bootstrap.domain`, `bootstrap.infrastructure.primary` e `bootstrap.infrastructure.secondary` não dependem de `bootstrap.composition`.
- [ ] A exceção não permite que outros packages criem composition roots arbitrárias.

### Milestone 4 - Verificar impacto e formatar

#### Goal

Garantir que o teste arquitetural esteja integrado ao ciclo normal do repositório.

#### Changes

- [ ] Ajustar imports, nomes de métodos e mensagens `because(...)` para ficarem claros.
- [ ] Garantir que o teste usa nomes expressivos e segue `.editorconfig`.
- [ ] Rodar formatter apenas se necessário e com cuidado para não alterar arquivos fora do escopo.

#### Validation

- [ ] Command: `./mvnw -Dtest=HexagonalArchTest test`
- [ ] Expected result: sucesso.
- [ ] Command: `./mvnw test`
- [ ] Expected result: sucesso em todos os testes unitários.
- [ ] Command: `npm run prettier:check`
- [ ] Expected result: sucesso sem arquivos fora de formato.
- [ ] Command: `./mvnw clean verify`
- [ ] Expected result: usuário deve rodar localmente quando o gate final for necessário e informar exit code/falhas relevantes.

#### Acceptance Criteria

- [ ] O novo teste roda no `./mvnw test`.
- [ ] O novo teste falharia em uma violação clara, como importar `FileSystemRuntimeModeConfigurationRepository` em `RuntimeExtensionApplicationService`.
- [ ] O plano registra qualquer exceção arquitetural mantida.

## Progress

- [x] Milestone 1 started
- [x] Milestone 1 completed
- [x] Milestone 2 started
- [x] Milestone 2 completed
- [x] Milestone 3 started
- [x] Milestone 3 completed
- [x] Milestone 4 started
- [x] Milestone 4 completed
- [x] Final validation requested from user

## Decisions

- Decision: Implementar o teste no `seed4j-cli`, não no projeto `seed4j`.
  Rationale: O objetivo é proteger a arquitetura do CLI, especialmente `bootstrap`, `command` e shared kernels.
  Date/Author: 2026-06-02 / Codex

- Decision: Usar ArchUnit `1.4.2`, a mesma versão usada pelo projeto `seed4j`.
  Rationale: Evita escolher versão arbitrária e mantém alinhamento entre os projetos relacionados.
  Date/Author: 2026-06-02 / Codex

- Decision: Não proibir `application` de depender de `domain`.
  Rationale: O `HexagonalArchTest` do `seed4j` só proíbe `application -> infrastructure`; exemplos como `BeersApplicationService` e `Seed4jsampleAuthorizations` dependem de domínio.
  Date/Author: 2026-06-02 / Codex

- Decision: Tratar `bootstrap.composition` como exceção explícita.
  Rationale: A composition root pre-Spring é necessária antes do container Spring existir e precisa montar primary, application, domain e secondary manualmente.
  Date/Author: 2026-06-02 / Codex

- Decision: Permitir que `command.infrastructure.primary` dependa de `bootstrap.domain`.
  Rationale: O estado atual da CLI usa o adapter primário de comandos para traduzir entrada e saída relacionadas ao runtime bootstrap, incluindo `RuntimeSelection`, `RuntimeExtensionInstallRequest` e `RuntimeExtensionInstallResult`. Refatorar essa integração para outro contrato está fora do escopo deste teste; a exceção fica limitada ao pacote primary do contexto command.
  Date/Author: 2026-06-02 / Codex

- Decision: Autorizar `org.yaml.snakeyaml..` e `ch.qos.logback.classic..` em `..domain..` por enquanto.
  Rationale: `RuntimeMetadata` lê metadata YAML e `Seed4JCliLauncher` ajusta logging de bootstrap em código atualmente localizado no domínio. Essas dependências são uma dívida arquitetural real, mas removê-las exigiria refatoração de fluxo fora do escopo do teste arquitetural inicial.
  Date/Author: 2026-06-02 / Codex

## Risks and Mitigations

- Risk: Copiar o `HexagonalArchTest` do `seed4j` sem adaptação pode reprovar por causa de `bootstrap.composition`.
  Mitigation: Criar regras diretas para as fronteiras principais e modelar composition root separadamente.

- Risk: O teste pode bloquear dependências legítimas entre `command.infrastructure.primary` e `bootstrap.application`.
  Mitigation: Inicialmente manter a mesma regra do `seed4j`, que proíbe dependência em domínio de outro bounded context, mas não proíbe uso de application de outro context.

- Risk: Adicionar ArchUnit pode afetar convergência de dependências.
  Mitigation: Usar a versão já adotada em `seed4j` e, se necessário, validar com `./mvnw -DskipTests enforcer:enforce@enforce-dependencyConvergence`.

- Risk: O teste pode ser permissivo demais e virar apenas uma cópia formal.
  Mitigation: Incluir pelo menos as regras que protegem `domain`, `application`, `primary`, `secondary`, shared kernels e composition root.

- Risk: `Class.forName(packageName)` em helpers de `package-info` pode ser frágil.
  Mitigation: Copiar o padrão já validado no `seed4j` e manter mensagens de erro claras em caso de falha.

- Risk: Exceções para SnakeYAML, Logback e `command.infrastructure.primary -> bootstrap.domain` podem cristalizar dívida arquitetural.
  Mitigation: Manter as exceções explícitas no teste e no ExecPlan para que futuras refatorações possam removê-las deliberadamente.

- Risk: `./mvnw test` pode apresentar falha intermitente ao carregar contexto Spring em testes existentes.
  Mitigation: Repetir a execução para confirmar se a falha é estável; nesta implementação, uma execução completa falhou uma vez em testes Spring existentes e a repetição `./mvnw -q test` passou.

## Validation Strategy

1. Rodar `./mvnw -Dtest=HexagonalArchTest test` após criar o teste.
2. Rodar `./mvnw test` para garantir que o novo teste integra ao suite unitário.
3. Rodar `npm run prettier:check` para validar formatação Java/XML.
4. Se houver suspeita de conflito de dependências, rodar `./mvnw -DskipTests enforcer:enforce@enforce-dependencyConvergence`.
5. Solicitar ao usuário o gate final `./mvnw clean verify` e pedir exit code com resumo de falhas, conforme diretriz do repositório.

## Rollout and Recovery

Rollout: entregar em um commit focado, por exemplo `test(architecture): enforce hexagonal boundaries`.

Recovery: se o teste bloquear desenvolvimento por falso positivo, ajustar a regra mais específica em `HexagonalArchTest` em vez de remover o teste inteiro.

Recovery: se a dependência ArchUnit causar conflito Maven, remover a dependência e a propriedade do `pom.xml`, depois reavaliar uma versão compatível.

Recovery: se `bootstrap.composition` gerar muitas exceções, manter apenas regras diretas de fronteira no primeiro commit e registrar uma decisão para endurecer composition root em um segundo commit.

## Lessons Learned

- O `seed4j-cli` já possui `@BusinessContext` e `@SharedKernel`, então está preparado para um teste similar ao `seed4j`.
- O `seed4j-cli` já possui `Seed4JCliHome`, confirmando a direção de transformar `user.home` em conceito de domínio.
- O projeto `seed4j` automatiza direção de dependência, mas não automatiza a convenção “application constrói serviço de domínio”.
- `bootstrap.composition` é uma necessidade arquitetural real do CLI e deve ser uma exceção nomeada, não um pacote livre.
- O teste focado revelou que `command.infrastructure.primary` depende de domínio do bootstrap para version output e instalação de runtime extension.
- O domínio do bootstrap ainda contém parsing YAML e ajuste de Logback; isso não é Spring/adapters, mas é uma impureza arquitetural que vale refatorar separadamente se o objetivo for domínio mais puro.
- `./mvnw test` passou uma vez com saída normal, falhou uma vez de forma intermitente em carregamento de contexto Spring existente, e passou novamente com `./mvnw -q test`; o novo `HexagonalArchTest` passou em todas as execuções.
