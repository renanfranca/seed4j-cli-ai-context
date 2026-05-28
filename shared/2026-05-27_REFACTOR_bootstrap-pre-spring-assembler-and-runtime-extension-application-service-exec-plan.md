# Refatorar bootstrap com assembler pré-Spring e application service único para runtime extension

This ExecPlan is a living document. Keep `Progress`, `Decisions`, `Risks`, and `Lessons Learned` up to date as work advances.

Safety boundary: This task is limited to authorized, defensive maintenance of this repository. Do not provide offensive guidance or policy-bypassing instructions.

## Purpose / Big Picture

O objetivo é tornar o bootstrap mais previsível para evolução sem quebrar o requisito central do CLI: parte do fluxo precisa acontecer antes do Spring Boot inicializar. Para o usuário final, o comportamento deve continuar igual no startup do CLI, e os comandos de extensão devem permanecer funcionais com mensagens e exit codes consistentes. O ganho principal é estrutural: separar claramente a composição pré-Spring da orquestração pós-Spring, reduzindo acoplamento entre adapters de entrada e detalhes de filesystem.

## Scope

In-scope:
- Extrair a composição pré-Spring para um assembler em `bootstrap/infrastructure/primary`.
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
- Assembler pré-Spring: classe de infraestrutura primária responsável por montar dependências concretas do launcher fora do container Spring.
- Porta: interface do domínio usada para inversão de dependência (exemplo: `RuntimeModeConfigurationRepository`).
- Adapter secondary: implementação técnica de porta (exemplo: `FileSystemRuntimeModeConfigurationRepository`).
- Application service de runtime extension: serviço da camada de aplicação que orquestra `install`, `enable` e `disable` sem conter regra de domínio complexa.

## Existing Context

O fluxo atual tem dois pontos de acoplamento relevantes:
1. `src/main/java/com/seed4j/cli/bootstrap/domain/Seed4JCliLauncherFactory.java` está no pacote `domain` e instancia `FileSystemRuntimeModeConfigurationRepository` (adapter secondary), criando dependência de domínio para infraestrutura.
2. `src/main/java/com/seed4j/cli/command/infrastructure/primary/ExtensionInstallCommand.java` instancia manualmente `RuntimeExtensionInstaller` e adapters de filesystem no construtor, misturando adapter primário com composição técnica.

O `Seed4JCliApp` já tem requisitos reais de pré-Spring em `productionBootstrapEntryPoint(...)`, então o plano preserva esse desenho e apenas melhora a fronteira arquitetural.

## Desired End State

- `Seed4JCliApp` delega montagem pré-Spring para um assembler dedicado em `bootstrap/infrastructure/primary`.
- `Seed4JCliLauncherFactory` permanece no domínio somente como factory de composição interna, sem importar classes de `bootstrap/infrastructure/secondary`.
- Um único `RuntimeExtensionApplicationService` existe em `bootstrap/application` com métodos para `install`, `enable` e `disable`.
- `RuntimeExtensionApplicationService` recebe interfaces (`RuntimeModeConfigurationRepository`, `RuntimeExtensionArtifactsRepository`) e `Path userHome`, e instancia internamente os serviços de domínio correspondentes.
- `ExtensionInstallCommand` deixa de instanciar adapters de filesystem e passa a depender do application service.
- Comportamento observável do usuário permanece compatível (startup, mensagens de erro e fluxo do comando install).

## Milestones

### Milestone 1 - Isolar composição pré-Spring no assembler primário

#### Goal

Separar a montagem técnica do launcher em uma classe explícita de pré-Spring sem mudar comportamento de execução do CLI.

#### Changes

Editar os seguintes arquivos:
1. Criar `src/main/java/com/seed4j/cli/bootstrap/infrastructure/primary/PreSpringLauncherAssembler.java` para montar:
   - `LocalSpringCliRunner`;
   - `ChildProcessLauncher`;
   - `RuntimeModeConfigurationRepository` (adapter filesystem);
   - `Seed4JCliLauncher` via `Seed4JCliLauncherFactory`.
2. Editar `src/main/java/com/seed4j/cli/Seed4JCliApp.java` para delegar `productionBootstrapEntryPoint(...)` ao novo assembler.
3. Editar `src/main/java/com/seed4j/cli/bootstrap/domain/Seed4JCliLauncherFactory.java` para remover import de adapter secondary e receber `RuntimeModeConfigurationRepository` como dependência de entrada.
4. Atualizar/ajustar testes de bootstrap afetados:
   - `src/test/java/com/seed4j/cli/Seed4JCliAppTest.java`;
   - `src/test/java/com/seed4j/cli/bootstrap/domain/Seed4JCliLauncherFactoryTest.java`.

#### Validation

Command: `./mvnw -Dtest=Seed4JCliAppTest,Seed4JCliLauncherFactoryTest,Seed4JCliLauncherTest test`  
Expected result: testes de bootstrap passam e o fluxo pré-Spring continua funcional sem regressões de exit code.

#### Acceptance Criteria

1. Não existe import de `bootstrap.infrastructure.secondary` em `Seed4JCliLauncherFactory`.
2. `Seed4JCliApp` não contém mais composição detalhada de launcher; ela é delegada ao assembler.
3. Execução de bootstrap preserva contratos atuais de fallback/local/child mode.

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
- [ ] Milestone 1 started
- [ ] Milestone 1 completed
- [ ] Milestone 2 started
- [ ] Milestone 2 completed
- [ ] Milestone 3 started
- [ ] Milestone 3 completed

## Decisions

- Decision: Manter `PreSpringLauncherAssembler` em `bootstrap/infrastructure/primary`.
  Rationale: o fluxo de entrada pré-Spring é um adapter primário do contexto bootstrap e precisa existir fora do container.
  Date/Author: 2026-05-27 / Renan + Codex

- Decision: Usar um único `RuntimeExtensionApplicationService` para `install`, `enable`, `disable`.
  Rationale: os três casos compartilham contexto e dependências; simplifica MVP sem impedir split futuro.
  Date/Author: 2026-05-27 / Renan + Codex

- Decision: Application service deve receber portas/interfaces, não classes concretas de domínio prontas.
  Rationale: alinhamento com o padrão usado em `seed4j/module/application/Seed4JModulesApplicationService`.
  Date/Author: 2026-05-27 / Renan + Codex

- Decision: Preservar `Seed4JCliLauncherFactory` no domínio, mas sem import de secondary.
  Rationale: manter construtor package-private do launcher e fronteira de pacote, removendo vazamento de camada.
  Date/Author: 2026-05-27 / Renan + Codex

## Risks and Mitigations

- Risk: regressão no startup pré-Spring ao mover composição para assembler.
  Mitigation: manter testes focados de `Seed4JCliApp` e `Seed4JCliLauncher` no primeiro milestone.

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

- A ser preenchido durante a execução.
