# ExecPlan: Refatorar `RuntimeExtensionInstaller` com boundary de persistência incremental

Este ExecPlan é um documento vivo. Durante execução, manter `Progress`, `Decisions`, `Risks` e `Lessons Learned` atualizados continuamente.

## Purpose / Big Picture

Vamos fazer um refactor incremental para separar regra de negócio de I/O no fluxo **já produtivo** de `extension install`, sem mexer ainda em `enabler/disabler`. O resultado visível para usuário final deve ser o mesmo (`seed4j extension install` com mesmas mensagens e efeitos), mas com `RuntimeExtensionInstaller` sem manipulação direta de arquivo. Isso reduz acoplamento, facilita teste unitário real de orquestração e prepara o próximo slice para `enabler/disabler`.

Safety boundary: This task is limited to authorized, defensive maintenance of this repository. Do not provide offensive guidance or policy-bypassing instructions.

## Scope

Em escopo:

- Extrair dependências de filesystem/configuração usadas por `RuntimeExtensionInstaller` para interfaces de domínio + implementações em `bootstrap/infrastructure/secondary`.
- Atualizar wiring do comando `extension install`.
- Manter comportamento funcional atual do installer.
- Validar o slice com testes focados no installer.

Fora de escopo (neste plano):

- Refactor de `RuntimeExtensionModeEnabler` e `RuntimeExtensionModeDisabler`.
- Introdução da camada `bootstrap/application`.
- Fechar, neste mesmo slice, a cobertura pendente de `enabler/disabler`.
- Decisão final sobre estratégia de cobertura de branches não determinísticos de I/O.

## Definitions

- **Boundary de persistência**: interface no domínio que representa operações de leitura/escrita, sem expor detalhes de `Files`, YAML ou move atômico.
- **Adapter secondary**: implementação concreta da interface de domínio usando filesystem local.
- **Slice incremental (installer-only)**: mudança limitada ao fluxo `extension install`, deixando toggles de modo para etapa posterior.
- **Equivalência comportamental**: mesma saída de erro/sucesso e mesmos arquivos gerados/substituídos para os cenários já cobertos.

## Existing Context

- `RuntimeExtensionInstaller` concentra regra e I/O (copy/move/write YAML).
- `ExtensionInstallCommand` instancia `RuntimeExtensionInstaller` diretamente.
- Existe duplicação de escrita/configuração com `RuntimeModeConfigurationWriter`.
- `RuntimeExtensionModeEnabler/Disabler` estão fora do fluxo de comando atual e continuam com cobertura pendente.
- Regra de cobertura do projeto é estrita por classe (`MISSEDCOUNT=0`), mas este plano foi explicitamente aprovado como **escopo estrito installer**.

## Desired End State

- `RuntimeExtensionInstaller` fica como orquestrador de domínio (valida entrada, decide fluxo, delega I/O a interfaces).
- Operações de filesystem e persistência de config usadas pelo installer ficam em `bootstrap/infrastructure/secondary`.
- `ExtensionInstallCommand` compõe o caso de uso com adapters secondary concretos.
- Testes do installer seguem verdes, sem regressão de comportamento observável.
- Diferenças de cobertura fora do escopo (enabler/disabler) ficam registradas como dívida explícita para próximo ExecPlan.

## Milestones

### Milestone 1 - Definir boundaries de domínio do installer

#### Goal

Definir contratos de persistência para o fluxo de instalação sem acoplamento a tecnologia.

#### Changes

- [x] Criar interfaces no domínio para o fluxo do installer:
  - `src/main/java/com/seed4j/cli/bootstrap/domain/RuntimeModeConfigurationRepository.java`
  - `src/main/java/com/seed4j/cli/bootstrap/domain/RuntimeExtensionArtifactsRepository.java`
- [x] Atualizar `RuntimeExtensionInstaller` para depender dessas interfaces via construtor explícito.
- [x] Manter `RuntimeExtensionJarLayoutValidator` e tipos de request/resultado no domínio.

#### Validation

- [x] Comando: `./mvnw -Dtest=RuntimeExtensionInstallerTest test`
- [x] Resultado esperado: testes do installer compilam e passam com novo contrato.

#### Acceptance Criteria

- [x] `RuntimeExtensionInstaller` não usa `Files`, `Yaml`, `StandardCopyOption` diretamente.
- [x] Fluxo de instalação permanece semanticamente igual para sucesso/erro já cobertos.

### Milestone 2 - Implementar adapters secondary e wiring do comando

#### Goal

Mover I/O real do installer para infraestrutura secondary com wiring explícito no primary command.

#### Changes

- [ ] Adicionar implementações filesystem:
  - `src/main/java/com/seed4j/cli/bootstrap/infrastructure/secondary/FileSystemRuntimeModeConfigurationRepository.java`
  - `src/main/java/com/seed4j/cli/bootstrap/infrastructure/secondary/FileSystemRuntimeExtensionArtifactsRepository.java`
- [ ] Atualizar `src/main/java/com/seed4j/cli/command/infrastructure/primary/ExtensionInstallCommand.java` para instanciar o installer com essas implementações.
- [ ] Garantir preservação de mensagens/erros de `InvalidRuntimeConfigurationException` já esperadas pelos testes existentes.

#### Validation

- [ ] Comando: `./mvnw -Dtest=RuntimeExtensionInstallerTest test`
- [ ] Resultado esperado: mesmos cenários de install (create/overwrite/failure) continuam verdes.

#### Acceptance Criteria

- [ ] I/O do installer está encapsulado nos adapters secondary.
- [ ] O comando `extension install` continua funcional sem alteração de contrato CLI.

### Milestone 3 - Consolidar testes e registrar dívida do próximo slice

#### Goal

Fechar validação do slice e documentar riscos pendentes fora do escopo.

#### Changes

- [ ] Ajustar testes do installer se necessário para novo construtor/wiring.
- [ ] Registrar no ExecPlan a dívida explícita: refactor de `enabler/disabler` e estratégia final de cobertura de branches não determinísticos.
- [ ] Não alterar ainda `RuntimeExtensionModeEnabler/Disabler`.

#### Validation

- [ ] Comando: `./mvnw -Dtest=RuntimeExtensionInstallerTest test`
- [ ] Resultado esperado: verde.
- [ ] Comando (observacional, não gate deste slice): `./mvnw clean verify`
- [ ] Resultado esperado: pode manter falhas preexistentes fora do escopo (enabler/disabler), sem novas regressões atribuíveis ao installer.

#### Acceptance Criteria

- [ ] Slice installer entregue com boundary + secondary operacional.
- [ ] Próximo passo fica claramente definido e rastreável.

## Progress

- [x] Milestone 1 started
- [x] Milestone 1 completed
- [ ] Milestone 2 started
- [ ] Milestone 2 completed
- [ ] Milestone 3 started
- [ ] Milestone 3 completed

## Decisions

- Decision: Usar naming sem sufixo `Port`, alinhado ao padrão `seed4j/module` (`*Repository`, `FileSystem*`).
  Rationale: Consistência arquitetural já adotada no ecossistema Seed4J.
  Date/Author: 2026-05-26 / Renan + Codex

- Decision: Executar refactor incremental com escopo inicial **somente installer**.
  Rationale: Reduz risco de big bang e entrega valor com menor superfície.
  Date/Author: 2026-05-26 / Renan + Codex

- Decision: Estratégia de cobertura para branches não determinísticos de I/O fica adiada para próximo slice.
  Rationale: Priorizar boundary e separação de responsabilidades no installer primeiro.
  Date/Author: 2026-05-26 / Renan + Codex

- Decision: Critério de aceite deste plano é escopo estrito installer (full `verify` pode permanecer com falhas preexistentes fora do escopo).
  Rationale: Decisão explícita de escopo para evitar expansão não planejada.
  Date/Author: 2026-05-26 / Renan + Codex

- Decision: Introduzir implementações filesystem default no domínio (`RuntimeModeConfigurationFileSystemRepository`, `RuntimeExtensionArtifactsFileSystemRepository`) como passo transitório para manter o command path sem alteração no Milestone 1.
  Rationale: Permitir injeção explícita de boundaries no installer sem antecipar o wiring do comando previsto para o Milestone 2.
  Date/Author: 2026-05-26 / Codex

## Risks and Mitigations

- Risk: Mudança de mensagens de erro pode quebrar testes/UX.
  Mitigation: Preservar mensagens atuais e validar cenários de erro existentes no `RuntimeExtensionInstallerTest`.

- Risk: Wiring manual no comando pode introduzir acoplamento inadequado.
  Mitigation: Limitar composição ao primary command e manter domínio dependente apenas de interfaces.

- Risk: Duplicação global não zera enquanto enabler/disabler não forem migrados.
  Mitigation: Registrar dívida como próximo ExecPlan obrigatório após concluir este slice.

- Risk: `clean verify` continuar falhando por classes fora do escopo pode confundir status.
  Mitigation: Documentar explicitamente que gate deste slice é teste focado do installer; `verify` completo é observacional neste plano.

## Validation Strategy

1. Rodar testes focados no installer:
   - `./mvnw -Dtest=RuntimeExtensionInstallerTest test`
2. (Opcional observacional) Rodar validação ampla:
   - `./mvnw clean verify`
3. Confirmar equivalência comportamental:
   - cenários de criação, sobrescrita e falhas já cobertos permanecem com mesmo resultado.

## Rollout and Recovery

- Rollout:
  - Entregar como refactor interno sem alteração de contrato CLI.
  - Revisar diff para garantir ausência de mudança funcional acidental nas mensagens e no resultado do comando.
- Recovery:
  - Em caso de regressão, reverter apenas commits do slice installer.
  - Manter fallback imediato para versão anterior do `RuntimeExtensionInstaller` enquanto se corrige adapter específico.

## Lessons Learned

- Separar apenas o fluxo produtivo (`installer`) acelera entrega, mas exige registrar dívida técnica explicitamente para não normalizar exceções arquiteturais.
- Em projeto com cobertura por classe estrita, “escopo funcional” e “gate de qualidade global” podem divergir; essa divergência precisa ficar formalizada no plano para evitar ruído operacional.
- Injeção explícita de boundary no construtor do installer permitiu validar orquestração via teste sem depender de I/O real, enquanto o caminho público do comando permaneceu estável.
