# Refatorar runtime-mode para boundary única e helpers package-private em secondary

Este ExecPlan é um documento vivo. Mantenha `Progress`, `Decisions`, `Risks` e `Lessons Learned` atualizados durante a execução.

Safety boundary: This task is limited to authorized, defensive maintenance of this repository. Do not provide offensive guidance or policy-bypassing instructions.

Arquivo sugerido para registrar este plano: `_temporary/ai_agent/seed4j-cli-ai-context/shared/2026-05-26_REFACTOR_runtime-mode-secondary-boundary-enabler-disabler-launcher-exec-plan.md`.

## Purpose / Big Picture

Este refactor remove vazamento de infraestrutura do domínio ao mover `RuntimeModeConfigReader` e `RuntimeModeConfigurationWriter` para `bootstrap/infrastructure/secondary` como helpers package-private. O efeito visível para usuário final deve permanecer igual em `seed4j extension install`, no comportamento de enable/disable (serviços de domínio) e no bootstrap do launcher. O ganho é arquitetural e operacional: um único boundary de configuração de runtime mode para installer, enabler/disabler e launcher, sem dependência direta do domínio em `Files`/YAML.

## Scope

In-scope:

- Generalizar o contrato de `RuntimeModeConfigurationRepository` para suportar leitura/persistência neutra de mode.
- Migrar `RuntimeExtensionInstaller`, `RuntimeExtensionModeEnabler`, `RuntimeExtensionModeDisabler` e `Seed4JCliLauncher` para depender do boundary.
- Mover `RuntimeModeConfigReader` e `RuntimeModeConfigurationWriter` para `bootstrap/infrastructure/secondary` como package-private.
- Remover as classes públicas equivalentes do pacote `bootstrap/domain`.
- Preservar mensagens de erro e semântica comportamental já validada por testes.

Out-of-scope:

- Criar novos comandos CLI (`extension enable/disable`) neste plano.
- Alterar formato de `~/.config/seed4j-cli/config.yml`.
- Alterar regras de bootstrap além da injeção do novo boundary.
- Introduzir `bootstrap/application` ou refactor amplo fora do runtime-mode.

## Definitions

- Runtime mode boundary: interface de domínio que abstrai leitura de configuração, leitura de mode e persistência de mode sem expor detalhes de filesystem/YAML.
- RuntimeModeConfigurationDocument: tipo de domínio para representar configuração runtime carregada e reutilizada nas gravações.
- Secondary helpers package-private: classes de apoio de I/O/YAML sem visibilidade pública fora de `bootstrap/infrastructure/secondary`.
- Equivalência comportamental: mesmos códigos de saída, mensagens e efeitos em arquivo para cenários já cobertos.

## Existing Context

- `RuntimeModeConfigReader` e `RuntimeModeConfigurationWriter` estão públicos em `src/main/java/com/seed4j/cli/bootstrap/domain`.
- `FileSystemRuntimeModeConfigurationRepository` já existe em `src/main/java/com/seed4j/cli/bootstrap/infrastructure/secondary`, mas importa os helpers públicos do domínio.
- `RuntimeExtensionInstaller` já depende de `RuntimeModeConfigurationRepository`, porém o contrato atual é enviesado para install (`persistExtensionMode`).
- `RuntimeExtensionModeEnabler`, `RuntimeExtensionModeDisabler` e `Seed4JCliLauncher` ainda instanciam `RuntimeModeConfigReader`/`RuntimeModeConfigurationWriter` diretamente.
- Planos base relacionados:

1. `_temporary/ai_agent/seed4j-cli-ai-context/shared/2026-05-26_REFACTOR_runtime-extension-installer-persistence-boundary-exec-plan.md`
2. `_temporary/ai_agent/seed4j-cli-ai-context/shared/2026-05-26_FEATURE_seed4j-extension-enable-disable-mvp-exec-plan.md`
3. Continuação concluída: `_temporary/ai_agent/seed4j-cli-ai-context/shared/2026-05-27_REFACTOR_runtime-mode-prepare-mode-change-protocol-exec-plan.md`

## Desired End State

- O domínio não expõe nem instancia `RuntimeModeConfigReader`/`RuntimeModeConfigurationWriter`.
- Existe um boundary de runtime-mode único e neutro para os três fluxos: installer, enabler/disabler e launcher.
- `RuntimeModeConfigReader` e `RuntimeModeConfigurationWriter` existem apenas em `bootstrap/infrastructure/secondary` com visibilidade package-private.
- Os testes de installer, enabler/disabler e launcher continuam verdes sem regressão funcional.

## Milestones

### Milestone 1 - Generalizar boundary de runtime mode e internalizar helpers em secondary

#### Goal

Definir o contrato de domínio neutro para runtime mode e mover implementação de parsing/writing para infraestrutura secondary sem alterar comportamento observável.

#### Changes

1. Editar `src/main/java/com/seed4j/cli/bootstrap/domain/RuntimeModeConfigurationRepository.java` para substituir API enviesada (`persistExtensionMode`) por API neutra de mode.
2. Criar `src/main/java/com/seed4j/cli/bootstrap/domain/RuntimeModeConfigurationDocument.java` como tipo dedicado para configuração carregada (TDD por tipo de negócio).
3. Criar `src/main/java/com/seed4j/cli/bootstrap/infrastructure/secondary/RuntimeModeConfigReader.java` package-private com a lógica atual de leitura/validação YAML.
4. Criar `src/main/java/com/seed4j/cli/bootstrap/infrastructure/secondary/RuntimeModeConfigurationWriter.java` package-private com a lógica atual de persistência YAML.
5. Atualizar `src/main/java/com/seed4j/cli/bootstrap/infrastructure/secondary/FileSystemRuntimeModeConfigurationRepository.java` para usar os novos helpers secondary e implementar a API neutra.
6. Preservar mensagens de exceção já esperadas nos testes (`Could not read ~/.config/seed4j-cli/config.yml.`, mensagens de YAML inválido).

#### Validation

1. Command: `./mvnw -Dtest=RuntimeExtensionInstallerTest test`
2. Expected result: suíte verde com os mesmos cenários de criação/sobrescrita/falha.
3. Command: `./mvnw -Dtest=RuntimeExtensionModeEnablerTest,RuntimeExtensionModeDisablerTest test`
4. Expected result: suÍtes verdes garantindo que leitura/escrita de config não mudou semanticamente.

#### Acceptance Criteria

1. Boundary neutro de runtime mode compilando e utilizado pelo adapter secondary.
2. Helpers de I/O/YAML não são mais públicos em domínio para novos usos.
3. Sem regressões nas mensagens de erro de configuração inválida.

### Milestone 2 - Migrar installer e serviços enable/disable para a nova porta

#### Goal

Concluir a migração do fluxo de instalação e dos serviços de alternância de modo para a API neutra do boundary, removendo dependência direta de helpers no domínio.

#### Changes

1. Editar `src/main/java/com/seed4j/cli/bootstrap/domain/RuntimeExtensionInstaller.java` para usar `readConfiguration`/`persistMode(..., RuntimeMode.EXTENSION)` da nova porta.
2. Editar `src/main/java/com/seed4j/cli/bootstrap/domain/RuntimeExtensionModeEnabler.java` para depender de `RuntimeModeConfigurationRepository` injetado e remover `new RuntimeModeConfigReader()`/`RuntimeModeConfigurationWriter`.
3. Editar `src/main/java/com/seed4j/cli/bootstrap/domain/RuntimeExtensionModeDisabler.java` para depender de `RuntimeModeConfigurationRepository` injetado e remover `new RuntimeModeConfigReader()`/`RuntimeModeConfigurationWriter`.
4. Ajustar wiring em `src/main/java/com/seed4j/cli/command/infrastructure/primary/ExtensionInstallCommand.java` para continuar compondo com `FileSystemRuntimeModeConfigurationRepository`.
5. Ajustar testes diretamente impactados:
   `src/test/java/com/seed4j/cli/bootstrap/domain/RuntimeExtensionInstallerTest.java`,
   `src/test/java/com/seed4j/cli/bootstrap/domain/RuntimeExtensionModeEnablerTest.java`,
   `src/test/java/com/seed4j/cli/bootstrap/domain/RuntimeExtensionModeDisablerTest.java`.

#### Validation

1. Command: `./mvnw -Dtest=RuntimeExtensionInstallerTest,RuntimeExtensionModeEnablerTest,RuntimeExtensionModeDisablerTest test`
2. Expected result: todas as suítes verdes.
3. Command: `./mvnw -Dtest=ExtensionInstallCommandTest test`
4. Expected result: sem regressão no comando `extension install` e mensagens de sucesso/erro preservadas.

#### Acceptance Criteria

1. Installer, enabler e disabler não instanciam classes de I/O/YAML diretamente.
2. Persistência de `mode: extension|standard` permanece com mesmo efeito funcional.
3. Contrato CLI de `extension install` permanece inalterado.

### Milestone 3 - Migrar launcher e remover helpers públicos antigos do domínio

#### Goal

Fechar o boundary em todo o bootstrap e eliminar definitivamente os helpers públicos antigos de domínio.

#### Changes

1. Editar `src/main/java/com/seed4j/cli/bootstrap/domain/Seed4JCliLauncher.java` para usar `RuntimeModeConfigurationRepository.readMode()` em vez de `RuntimeModeConfigReader`.
2. Editar `src/main/java/com/seed4j/cli/bootstrap/domain/Seed4JCliLauncherFactory.java` para compor `Seed4JCliLauncher` com `FileSystemRuntimeModeConfigurationRepository`.
3. Remover `src/main/java/com/seed4j/cli/bootstrap/domain/RuntimeModeConfigReader.java`.
4. Remover `src/main/java/com/seed4j/cli/bootstrap/domain/RuntimeModeConfigurationWriter.java`.
5. Ajustar testes de launcher que constroem o objeto diretamente:
   `src/test/java/com/seed4j/cli/bootstrap/domain/Seed4JCliLauncherTest.java`,
   `src/test/java/com/seed4j/cli/bootstrap/domain/ExtensionRuntimeBootstrapInProcessTest.java`.

#### Validation

1. Command: `./mvnw -Dtest=Seed4JCliLauncherTest,ExtensionRuntimeBootstrapInProcessTest test`
2. Expected result: suÍtes verdes, incluindo cenários de extension-mode/standard-mode e validações de bootstrap.
3. Command: `./mvnw -Dtest=RuntimeExtensionInstallerTest,RuntimeExtensionModeEnablerTest,RuntimeExtensionModeDisablerTest,ExtensionInstallCommandTest test`
4. Expected result: regressão cruzada inexistente após remoção dos helpers antigos.

#### Acceptance Criteria

1. Nenhuma classe em `bootstrap/domain` realiza leitura/escrita YAML de runtime mode.
2. `RuntimeModeConfigReader` e `RuntimeModeConfigurationWriter` existem apenas em `bootstrap/infrastructure/secondary` com visibilidade package-private.
3. Fluxos de launcher, install e toggle mantêm equivalência comportamental.

### Milestone 4 - Consolidação final e atualização dos planos relacionados

#### Goal

Consolidar validação ampla e registrar dívida remanescente/decisões para os próximos slices.

#### Changes

1. Atualizar este ExecPlan com status final de milestones, decisões efetivas e aprendizados.
2. Atualizar referência no plano de enable/disable MVP para indicar que o refactor arquitetural de runtime-mode foi concluído.
3. Confirmar que não houve mudança de contrato público de comandos além do comportamento já existente.

#### Validation

1. Command: `./mvnw clean verify`
2. Expected result: validação ampla do repositório sem regressões atribuíveis ao refactor.
3. Command: `npm run prettier:check`
4. Expected result: sem divergências de formatação.

#### Acceptance Criteria

1. ExecPlan atualizado como documento vivo e handoff-safe.
2. Qualidade validada do nível focado ao amplo.
3. Próximo passo arquitetural documentado sem ambiguidade.

## Progress

- [x] Milestone 1 started
- [x] Milestone 1 completed
- [x] Milestone 2 started
- [x] Milestone 2 completed
- [x] Milestone 3 started
- [x] Milestone 3 completed
- [x] Milestone 4 started
- [x] Milestone 4 completed

### Milestone 1 Execution Notes (2026-05-26)

- Boundary generalizado em `RuntimeModeConfigurationRepository` com API neutra:
  - `readConfiguration()`
  - `readMode()`
  - `persistMode(..., RuntimeMode mode)`
- Novo tipo de domínio criado: `RuntimeModeConfigurationDocument`.
- Helpers de leitura/escrita YAML criados em `bootstrap/infrastructure/secondary` como package-private:
  - `RuntimeModeConfigReader`
  - `RuntimeModeConfigurationWriter`
- `FileSystemRuntimeModeConfigurationRepository` migrou para os helpers secondary e para a API neutra.
- `RuntimeExtensionInstaller` passou a persistir via `persistMode(..., RuntimeMode.EXTENSION)` usando o documento tipado.
- Validação executada:
  - `./mvnw -Dtest=RuntimeExtensionInstallerTest test`
  - `./mvnw -Dtest=RuntimeExtensionModeEnablerTest,RuntimeExtensionModeDisablerTest test`
  - Checkpoint vertical extra: `./mvnw -Dtest=ExtensionInstallCommandTest test`

### Milestone 2 Execution Notes (2026-05-27)

- `RuntimeExtensionInstaller` já estava usando `readConfiguration` + `persistMode(..., RuntimeMode.EXTENSION)` desde o milestone 1.
- `RuntimeExtensionModeDisabler` migrou para depender de `RuntimeModeConfigurationRepository` injetado e persistir via `persistMode(..., RuntimeMode.STANDARD)`.
- `RuntimeExtensionModeEnabler` migrou para depender de `RuntimeModeConfigurationRepository` injetado e persistir via `persistMode(..., RuntimeMode.EXTENSION)`.
- `RuntimeExtensionModeEnabler` manteve `userHome` apenas para validar runtime extension ativo (`RuntimeSelection.resolve`) antes da persistência do modo.
- `ExtensionInstallCommand` permaneceu compondo com `FileSystemRuntimeModeConfigurationRepository` (sem alteração funcional de wiring necessária neste slice).
- Validação executada:
  - `./mvnw -Dtest=RuntimeExtensionInstallerTest,RuntimeExtensionModeEnablerTest,RuntimeExtensionModeDisablerTest test`
  - Checkpoint vertical: `./mvnw -Dtest=ExtensionInstallCommandTest test`

### Milestone 3 Execution Notes (2026-05-27)

- `Seed4JCliLauncher` passou a depender explicitamente de `RuntimeModeConfigurationRepository` e usa `readMode()` como única fonte de verdade para runtime mode.
- `Seed4JCliLauncherFactory` passou a compor `Seed4JCliLauncher` com `FileSystemRuntimeModeConfigurationRepository`, removendo fallback de leitura de YAML dentro do launcher.
- Testes que instanciavam launcher diretamente foram migrados para o novo construtor com porta injetada:
  - `Seed4JCliLauncherTest`
  - `ExtensionRuntimeBootstrapInProcessTest`
- Os helpers legados foram removidos de `bootstrap/domain`:
  - `RuntimeModeConfigReader.java`
  - `RuntimeModeConfigurationWriter.java`
- Foi adicionado teste explícito em `Seed4JCliLauncherFactoryTest` para garantir:
  - ausência do construtor legado sem repositório;
  - ausência das classes legadas de YAML no pacote de domínio.
- Validação executada:
  - `./mvnw -Dtest=Seed4JCliLauncherTest,Seed4JCliLauncherFactoryTest,ExtensionRuntimeBootstrapInProcessTest test`
  - Checkpoint vertical: `./mvnw -Dtest=Seed4JCliAppTest test`
  - Regressão cruzada do milestone: `./mvnw -Dtest=RuntimeExtensionInstallerTest,RuntimeExtensionModeEnablerTest,RuntimeExtensionModeDisablerTest,ExtensionInstallCommandTest test`

### Milestone 4 Execution Notes (2026-05-27)

- ExecPlan principal atualizado com fechamento do refactor de runtime-mode em launcher/install/toggle.
- Plano relacionado de enable/disable MVP atualizado para registrar que o refactor arquitetural de runtime-mode foi concluído:
  - `_temporary/ai_agent/seed4j-cli-ai-context/shared/2026-05-26_FEATURE_seed4j-extension-enable-disable-mvp-exec-plan.md`.
- Durante `./mvnw clean verify`, o gate de cobertura apontou lacunas em caminhos de erro/fallback:
  - `RuntimeModeConfigurationWriter` (fallback `AtomicMoveNotSupportedException` e limpeza de arquivo temporário).
  - `RuntimeExtensionModeEnabler` e `RuntimeExtensionModeDisabler` (tratamento de `IOException` de persistência).
- Cobertura foi fechada com testes adicionais mínimos:
  - `RuntimeModeConfigurationWriterTest` (novo).
  - Cenários de falha de persistência em `RuntimeExtensionModeEnablerTest` e `RuntimeExtensionModeDisablerTest`.
- Contrato público de comandos permaneceu inalterado neste slice (nenhum comando novo/alterado além do comportamento já existente).
- Validação executada:
  - `./mvnw -Dtest=RuntimeExtensionModeEnablerTest,RuntimeExtensionModeDisablerTest,RuntimeModeConfigurationWriterTest test`
  - `./mvnw clean verify`
  - `npm run prettier:check`

## Decisions

- Decision: Escopo completo incluindo installer, enabler/disabler e launcher.
  Rationale: evitar meia-migração que mantém vazamento de infraestrutura no domínio.
  Date/Author: 2026-05-26 / Renan + Codex

- Decision: Generalizar agora a porta `RuntimeModeConfigurationRepository`.
  Rationale: contrato único evita duplicação e drift de semântica entre install/enable/disable/launcher.
  Date/Author: 2026-05-26 / Renan + Codex

- Decision: Registrar em novo arquivo de ExecPlan em `shared`.
  Rationale: manter rastreabilidade limpa sem reabrir planos já encerrados.
  Date/Author: 2026-05-26 / Renan + Codex

- Decision: `RuntimeModeConfigReader` e `RuntimeModeConfigurationWriter` devem existir apenas em `secondary` como package-private.
  Rationale: são detalhes de tecnologia (filesystem + YAML), não API de domínio.
  Date/Author: 2026-05-26 / Renan + Codex

- Decision: manter temporariamente as classes homônimas em `bootstrap/domain` até concluir migração de enabler/disabler/launcher no milestone 3.
  Rationale: evitar quebra comportamental nos fluxos que ainda instanciam os helpers legados enquanto o boundary neutro é adotado incrementalmente.
  Date/Author: 2026-05-26 / Codex

- Decision: construtores de `RuntimeExtensionModeEnabler` e `RuntimeExtensionModeDisabler` passam a exigir `RuntimeModeConfigurationRepository` injetado.
  Rationale: remover instanciação direta de helpers de I/O/YAML no domínio e centralizar persistência na porta neutra.
  Date/Author: 2026-05-27 / Codex

- Decision: `Seed4JCliLauncher` não mantém mais construtor legado sem `RuntimeModeConfigurationRepository`.
  Rationale: impedir recidiva de acoplamento do domínio com leitura YAML/Filesystem e forçar composição no boundary secundário.
  Date/Author: 2026-05-27 / Codex

- Decision: Cobrir explicitamente caminhos de fallback/erro de persistência para satisfazer gate de cobertura integral do módulo.
  Rationale: `clean verify` exige cobertura completa das classes impactadas, incluindo fluxos de exceção/fallback.
  Date/Author: 2026-05-27 / Codex

## Risks and Mitigations

- Risk: mudança de assinatura da porta quebrar vários testes de uma vez.
  Mitigation: executar milestones incrementais com suíte focada por fluxo a cada etapa.

- Risk: regressão de mensagens de erro em config inválida.
  Mitigation: manter asserts textuais existentes e incluir validação explícita de mensagens nos testes focados.

- Risk: refactor do launcher alterar fallback local/child-process sem perceber.
  Mitigation: manter `Seed4JCliLauncherTest` e `ExtensionRuntimeBootstrapInProcessTest` como gate obrigatório antes de avançar milestone.

- Risk: introduzir acoplamento de infraestrutura no domínio por construtores de conveniência.
  Mitigation: toda composição de adapters fica em factory/command; domínio recebe interfaces.

## Validation Strategy

1. Validar cada milestone com suíte focada do fluxo alterado.
2. Rodar suíte cruzada após remoções finais de classes antigas.
3. Rodar `./mvnw clean verify` como validação ampla.
4. Rodar `npm run prettier:check`.
5. Confirmar manualmente, quando aplicável, que `extension install` mantém saída e efeito já esperados pelos testes.

## Rollout and Recovery

Rollout:

1. Entregar como refactor interno sem alterar contrato público de CLI.
2. Revisar diff final para confirmar ausência de mudança funcional involuntária.

Recovery:

1. Reverter somente commits do milestone com regressão.
2. Se regressão ocorrer no launcher, priorizar rollback do wiring do launcher mantendo boundary de installer já estável.
3. Manter referência aos planos anteriores para reaplicar a migração incrementalmente sem retrabalho.

## Lessons Learned

- O boundary de runtime mode precisa ser único para evitar inconsistência entre install, toggle e bootstrap.
- Tornar helpers de I/O/YAML públicos no domínio acelera curto prazo, mas aumenta custo de evolução e risco de acoplamento estrutural.
- Quebra de compilação guiada por TDD é útil para expor pontos de acoplamento escondidos cedo no slice.
- Encapsular a configuração carregada em `RuntimeModeConfigurationDocument` reduziu acoplamento em `Map` cru sem alterar semântica de erro observável.
- Em `enabler`, manter `userHome` separado da porta de configuração evita misturar validação de runtime ativo com persistência de mode.
