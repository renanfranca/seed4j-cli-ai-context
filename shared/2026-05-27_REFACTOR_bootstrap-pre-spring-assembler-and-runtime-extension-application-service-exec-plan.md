# Refatorar bootstrap com orquestrador pré-Spring em application e assembler primário

This ExecPlan is a living document. Keep `Progress`, `Decisions`, `Risks`, and `Lessons Learned` up to date as work advances.

Safety boundary: This task is limited to authorized, defensive maintenance of this repository. Do not provide offensive guidance or policy-bypassing instructions.

## Purpose / Big Picture

O objetivo é tornar o bootstrap mais previsível para evolução sem quebrar o requisito central do CLI: parte do fluxo precisa acontecer antes do Spring Boot inicializar. Para o usuário final, o comportamento deve continuar igual no startup do CLI, e os comandos de extensão devem permanecer funcionais com mensagens e exit codes consistentes. O ganho principal é estrutural: separar claramente, no fluxo pré-Spring, a orquestração de caso de uso (application) da composição técnica (assembler), além de manter a orquestração pós-Spring de runtime extension em application.

## Scope

In-scope:

- Introduzir um orquestrador pré-Spring em `bootstrap/application` para o fluxo de bootstrap.
- Manter `PreSpringLauncherAssembler` em `bootstrap/infrastructure/primary` com papel estrito de composição técnica.
- Manter o construtor package-private de `Seed4JCliLauncher` e preservar a factory de domínio como ponto de criação interno ao pacote.
- Remover dependência `domain -> infrastructure.secondary` no caminho do launcher.
- Introduzir `bootstrap/application/RuntimeExtensionApplicationService` único para `install`, `enable` e `disable`.
- Fazer o application service receber portas (interfaces) e valores técnicos, criando internamente serviços de domínio.
- Atualizar wiring Spring para que comandos não instanciem adapters de filesystem diretamente.
- Refatorar `ExtensionInstallCommand` para depender do application service.

Out-of-scope:

- Alterar contrato funcional de runtime mode no launcher.
- Introduzir bypass de validação no bootstrap.
- Redesenhar todos os comandos de `command/infrastructure/primary` além do fluxo de extensão.
- Mudar formato do arquivo `~/.config/seed4j-cli/config.yml`.

## Definitions

- Pré-Spring: fluxo executado em `main(...)` antes de `SpringApplicationBuilder.run(...)`.
- Orquestrador pré-Spring: serviço de application que executa o caso de uso de bootstrap antes do contexto Spring existir.
- Assembler pré-Spring: classe de infraestrutura primária responsável por montar dependências concretas do launcher fora do container Spring.
- Porta: interface do domínio usada para inversão de dependência (exemplo: `RuntimeModeConfigurationRepository`).
- Adapter secondary: implementação técnica de porta (exemplo: `FileSystemRuntimeModeConfigurationRepository`).
- Application service de runtime extension: serviço da camada de aplicação que orquestra `install`, `enable` e `disable` sem conter regra de domínio complexa.

## Existing Context

O fluxo atual tem dois pontos de acoplamento relevantes:

1. `src/main/java/com/seed4j/cli/bootstrap/domain/Seed4JCliLauncherFactory.java` está no pacote `domain` e instancia `FileSystemRuntimeModeConfigurationRepository` (adapter secondary), criando dependência de domínio para infraestrutura.
2. `src/main/java/com/seed4j/cli/command/infrastructure/primary/ExtensionInstallCommand.java` instancia manualmente `RuntimeExtensionInstaller` e adapters de filesystem no construtor, misturando adapter primário com composição técnica.
3. O risco arquitetural discutido é `PreSpringLauncherAssembler` virar orquestrador de caso de uso; a decisão é manter orquestração no pacote `bootstrap/application`.

O `Seed4JCliApp` já tem requisitos reais de pré-Spring em `productionBootstrapEntryPoint(...)`, então o plano preserva esse desenho e apenas melhora a fronteira arquitetural.

## Desired End State

- `Seed4JCliApp` delega o caso de uso pré-Spring para um orquestrador em `bootstrap/application`.
- `PreSpringLauncherAssembler` permanece em `bootstrap/infrastructure/primary` e apenas monta dependências concretas desse orquestrador (sem decidir fluxo).
- `Seed4JCliLauncherFactory` permanece no domínio somente como factory de composição interna, sem importar classes de `bootstrap/infrastructure/secondary`.
- Um único `RuntimeExtensionApplicationService` existe em `bootstrap/application` com métodos para `install`, `enable` e `disable`.
- `RuntimeExtensionApplicationService` recebe interfaces (`RuntimeModeConfigurationRepository`, `RuntimeExtensionArtifactsRepository`) e `Path userHome`, e instancia internamente os serviços de domínio correspondentes.
- `ExtensionInstallCommand` deixa de instanciar adapters de filesystem e passa a depender do application service.
- Comportamento observável do usuário permanece compatível (startup, mensagens de erro e fluxo do comando install).

## Milestones

### Milestone 1 - Introduzir orquestrador pré-Spring e restringir assembler à composição

#### Goal

Separar explicitamente a orquestração pré-Spring (application) da composição técnica (assembler) sem mudar comportamento de execução do CLI.

#### Changes

Editar os seguintes arquivos:

1. Criar `src/main/java/com/seed4j/cli/bootstrap/application/PreSpringBootstrapApplicationService.java` para orquestrar o caso de uso pré-Spring (launch path e retorno de exit code).
2. Criar/editar `src/main/java/com/seed4j/cli/bootstrap/infrastructure/primary/PreSpringLauncherAssembler.java` para montar:
   - `LocalSpringCliRunner`;
   - `ChildProcessLauncher`;
   - `RuntimeModeConfigurationRepository` (adapter filesystem);
   - `Seed4JCliLauncher` via `Seed4JCliLauncherFactory`;
   - instância do `PreSpringBootstrapApplicationService`.
3. Editar `src/main/java/com/seed4j/cli/Seed4JCliApp.java` para delegar `productionBootstrapEntryPoint(...)` ao `PreSpringBootstrapApplicationService` montado pelo assembler.
4. Editar `src/main/java/com/seed4j/cli/bootstrap/domain/Seed4JCliLauncherFactory.java` para remover import de adapter secondary e receber `RuntimeModeConfigurationRepository` como dependência de entrada.
5. Atualizar/ajustar testes de bootstrap afetados:
   - `src/test/java/com/seed4j/cli/Seed4JCliAppTest.java`;
   - `src/test/java/com/seed4j/cli/bootstrap/application/PreSpringBootstrapApplicationServiceTest.java` (novo arquivo, se necessário);
   - `src/test/java/com/seed4j/cli/bootstrap/domain/Seed4JCliLauncherFactoryTest.java`.

#### Validation

Command: `./mvnw -Dtest=Seed4JCliAppTest,Seed4JCliLauncherFactoryTest,Seed4JCliLauncherTest test`  
Expected result: testes de bootstrap passam e o fluxo pré-Spring continua funcional sem regressões de exit code.

Command: `./mvnw -Dtest=PreSpringBootstrapApplicationServiceTest test`  
Expected result: orquestração pré-Spring comprovada em teste dedicado sem lógica de composição no assembler.

#### Acceptance Criteria

1. Não existe import de `bootstrap.infrastructure.secondary` em `Seed4JCliLauncherFactory`.
2. `PreSpringLauncherAssembler` monta dependências, mas não contém decisão de fluxo de negócio de bootstrap.
3. `Seed4JCliApp` não contém mais composição detalhada de launcher; ela é delegada ao fluxo `assembler -> application service`.
4. Execução de bootstrap preserva contratos atuais de fallback/local/child mode.

### Milestone 2 - Introduzir application service único para runtime extension

#### Goal

Concentrar a orquestração de `install`, `enable` e `disable` na camada `application`, recebendo portas e montando serviços de domínio internamente.

#### Changes

Editar os seguintes arquivos:

1. Criar `src/main/java/com/seed4j/cli/bootstrap/application/RuntimeExtensionApplicationService.java` com:
   - método `install(RuntimeExtensionInstallRequest request)`;
   - método `enable()`;
   - método `disable()`;
   - construtor recebendo `Path userHome`, `RuntimeModeConfigurationRepository`, `RuntimeExtensionArtifactsRepository`.
2. Criar `src/main/java/com/seed4j/cli/bootstrap/infrastructure/secondary/BootstrapRuntimeConfiguration.java` com beans para:
   - `RuntimeModeConfigurationRepository`;
   - `RuntimeExtensionArtifactsRepository`;
   - `RuntimeExtensionApplicationService`.
3. Editar `src/main/java/com/seed4j/cli/command/infrastructure/primary/ExtensionInstallCommand.java` para injetar `RuntimeExtensionApplicationService` e remover instâncias diretas de adapters/installer.
4. Ajustar testes de comando impactados:
   - `src/test/java/com/seed4j/cli/command/infrastructure/primary/ExtensionInstallCommandTest.java`.

#### Validation

Command: `./mvnw -Dtest=ExtensionInstallCommandTest,RuntimeExtensionInstallerTest,RuntimeExtensionModeEnablerTest,RuntimeExtensionModeDisablerTest test`  
Expected result: `ExtensionInstallCommand` continua com mesmos resultados observáveis e domínio continua cobrindo regras de install/enable/disable.

#### Acceptance Criteria

1. `ExtensionInstallCommand` não contém `new FileSystemRuntimeModeConfigurationRepository(...)` nem `new FileSystemRuntimeExtensionArtifactsRepository()`.
2. `RuntimeExtensionApplicationService` recebe portas e monta classes de domínio internamente.
3. Fluxo de `install` preserva mensagens e exit code de sucesso/erro já esperados.

### Milestone 3 - Consolidação arquitetural e validação ampla

#### Goal

Fechar o refactor com validação ampla, documentação arquitetural mínima e plano atualizado para handoff.

#### Changes

Editar os seguintes arquivos:

1. Atualizar este ExecPlan com estado final de progresso, decisões, riscos e aprendizados.
2. Se necessário, atualizar documentação arquitetural:
   - `documentation/Commands.md` (somente se output/fluxo observável mudar);
   - `README.md` (somente se descoberta de comportamento precisar ajuste).
3. Revisar imports e dependências para garantir ausência de novos vazamentos de camada no bootstrap.

#### Validation

Command: `./mvnw clean verify`  
Expected result: suíte completa, cobertura, checkstyle e integração passam.

Command: `npm run prettier:check`  
Expected result: sem divergências de formatação.

#### Acceptance Criteria

1. Pipeline local completo passa.
2. Não há dependência `domain -> infrastructure.secondary` no bootstrap.
3. Documento de execução reflete exatamente o estado final implementado.

## Progress

- [x] ExecPlan drafted
- [x] Milestone 1 started
- [x] Milestone 1 completed
- [ ] Milestone 2 started
- [ ] Milestone 2 completed
- [ ] Milestone 3 started
- [ ] Milestone 3 completed

Milestone 1 execution notes (2026-05-28):

- `PreSpringBootstrapApplicationService` criado em `bootstrap/application` para orquestrar `launch(args)` com `childMode`.
- `PreSpringLauncherAssembler` criado em `bootstrap/infrastructure/primary` para composição técnica de launcher pré-Spring.
- `Seed4JCliApp.productionBootstrapEntryPoint(...)` passou a delegar para `assembler -> application service`.
- `Seed4JCliLauncherFactory` removido de dependência direta de adapter secondary; agora recebe `RuntimeModeConfigurationRepository` por entrada.
- Validações executadas e aprovadas:
  - `./mvnw -Dtest=Seed4JCliAppTest,Seed4JCliLauncherFactoryTest,Seed4JCliLauncherTest test`
  - `./mvnw -Dtest=PreSpringBootstrapApplicationServiceTest test`
  - `seed4j --version` (exit code 0)

## Decisions

- Decision: Manter `PreSpringLauncherAssembler` em `bootstrap/infrastructure/primary`.
  Rationale: o fluxo de entrada pré-Spring é um adapter primário do contexto bootstrap e precisa existir fora do container.
  Date/Author: 2026-05-27 / Renan + Codex

- Decision: Introduzir `PreSpringBootstrapApplicationService` em `bootstrap/application` para orquestrar o caso de uso pré-Spring.
  Rationale: evitar que o assembler acumule regras de fluxo; assembler permanece composição técnica.
  Date/Author: 2026-05-28 / Renan + Codex

- Decision: Usar um único `RuntimeExtensionApplicationService` para `install`, `enable`, `disable`.
  Rationale: os três casos compartilham contexto e dependências; simplifica MVP sem impedir split futuro.
  Date/Author: 2026-05-27 / Renan + Codex

- Decision: Application service deve receber portas/interfaces, não classes concretas de domínio prontas.
  Rationale: alinhamento com o padrão usado em `seed4j/module/application/Seed4JModulesApplicationService`.
  Date/Author: 2026-05-27 / Renan + Codex

- Decision: Preservar `Seed4JCliLauncherFactory` no domínio, mas sem import de secondary.
  Rationale: manter construtor package-private do launcher e fronteira de pacote, removendo vazamento de camada.
  Date/Author: 2026-05-27 / Renan + Codex

- Decision: Expor seam package-private em `Seed4JCliApp.productionBootstrapEntryPoint(...)` para injeção de factory do `PreSpringBootstrapApplicationService` em teste.
  Rationale: validar delegação app -> assembler/application sem acoplar teste à composição interna nem quebrar encapsulamento público.
  Date/Author: 2026-05-28 / Renan + Codex

## Risks and Mitigations

- Risk: regressão no startup pré-Spring ao mover composição para assembler.
  Mitigation: manter testes focados de `Seed4JCliApp` e `Seed4JCliLauncher` no primeiro milestone.

- Risk: `PreSpringLauncherAssembler` voltar a acumular lógica de orquestração em refactors futuros.
  Mitigation: cobrir fluxo de orquestração em teste de `PreSpringBootstrapApplicationService` e revisar assembler para manter escopo de composição.

- Risk: duplicidade de composição entre assembler pré-Spring e configuração Spring pós-Spring.
  Mitigation: explicitar responsabilidades de cada trilha no código e no ExecPlan; evitar copiar regras de domínio.

- Risk: mudanças em `ExtensionInstallCommand` alterarem mensagens esperadas por testes/scripts.
  Mitigation: preservar texto existente de sucesso/erro e validar por testes atuais do comando.

- Risk: crescimento excessivo do `RuntimeExtensionApplicationService`.
  Mitigation: definir critério de split futuro (divergência de regras/dependências por método) e registrar na seção de lições.

## Validation Strategy

1. Rodar testes focados de bootstrap pré-Spring após Milestone 1.
2. Rodar testes focados de comandos e domínio de runtime extension após Milestone 2.
3. Rodar validação completa com `./mvnw clean verify`.
4. Rodar `npm run prettier:check`.
5. Confirmar cenário manual mínimo:
   - `seed4j --version` continua funcionando;
   - `seed4j extension install ...` mantém comportamento observado antes do refactor.

## Rollout and Recovery

Rollout:

1. Entregar como refactor interno sem mudança intencional de contrato público do CLI.
2. Priorizar merge após validação completa local.

Recovery:

1. Se regressão ocorrer no startup, reverter Milestone 1 isoladamente para restabelecer fluxo pré-Spring antigo.
2. Se regressão ocorrer no comando `extension install`, reverter Milestone 2 mantendo Milestone 1.
3. Revalidar com `./mvnw clean verify` após rollback parcial.

## Lessons Learned

- A separação `assembler -> application service` no pré-Spring preservou o comportamento público sem exigir mudanças no contrato de `runProductionPath`.
- Mover a dependência de `RuntimeModeConfigurationRepository` para a entrada do factory do launcher eliminou vazamento `domain -> infrastructure.secondary` com impacto pequeno na API.
- Um seam package-private para factory no app simplificou testes de delegação sem expor novas APIs públicas do CLI.
