# Introduzir protocolo prepare/apply para runtime mode sem regressão comportamental

This ExecPlan is a living document. Keep `Progress`, `Decisions`, `Risks`, and `Lessons Learned` up to date as work advances.

Safety boundary: This task is limited to authorized, defensive maintenance of this repository. Do not provide offensive guidance or policy-bypassing instructions.

## Purpose / Big Picture

Este plano reduz acoplamento temporal implícito no fluxo de runtime mode ao trocar a sequência solta `readConfiguration() -> persistMode(...)` por um protocolo explícito `prepareModeChange(...) -> apply()`. Para o usuário final, o comportamento dos fluxos `extension install`, enable e disable deve permanecer igual, com as mesmas mensagens de erro e a mesma persistência de `mode`. O ganho é confiabilidade de design: a ordem obrigatória de passos fica explícita na API de domínio e não espalhada em serviços.

## Scope

In-scope:
- Evoluir a porta `RuntimeModeConfigurationRepository` para o protocolo aprovado na conversa.
- Introduzir tipo de domínio `RuntimeModeChangePlan` com `apply()`.
- Migrar `RuntimeExtensionInstaller`, `RuntimeExtensionModeEnabler` e `RuntimeExtensionModeDisabler` para o novo protocolo.
- Atualizar o adapter `FileSystemRuntimeModeConfigurationRepository` para implementar o novo contrato mantendo validações e mensagens atuais.
- Atualizar testes focados em ordem e falhas para evitar regressão de fail-fast.

Out-of-scope:
- Lock de arquivo entre processos.
- Garantias transacionais distribuídas entre artefatos e config.
- Mudanças no formato `~/.config/seed4j-cli/config.yml`.
- Mudança de comportamento funcional de launcher além de compatibilidade de compilação.

## Definitions

- Acoplamento temporal implícito: quando a ordem correta dos passos existe, mas fica escondida em chamadas soltas sem tipo/protocolo dedicado.
- Runtime mode change plan: objeto de domínio retornado por `prepareModeChange(...)` que encapsula snapshot validado da configuração e expõe `apply()` para persistência do modo alvo.
- Regra de domínio de ordenação: política "somente aplicar mode extension após pré-condições e instalação" definida na orquestração de domínio.
- Mecânica secondary: detalhes de I/O/YAML (read, validation, write atômico best-effort) implementados no adapter.

## Existing Context

Contexto atual no repositório:
- A porta está em `src/main/java/com/seed4j/cli/bootstrap/domain/RuntimeModeConfigurationRepository.java`.
- O domínio atualmente usa `RuntimeModeConfigurationDocument` como snapshot de config.
- Os serviços `RuntimeExtensionInstaller`, `RuntimeExtensionModeEnabler` e `RuntimeExtensionModeDisabler` ainda dependem da sequência explícita `readConfiguration()` e depois `persistMode(...)`.
- O adapter filesystem está em `src/main/java/com/seed4j/cli/bootstrap/infrastructure/secondary/FileSystemRuntimeModeConfigurationRepository.java` e já centraliza parse/write via helpers secondary.
- Este plano continua a trilha arquitetural do plano base: `_temporary/ai_agent/seed4j-cli-ai-context/shared/2026-05-26_REFACTOR_runtime-mode-secondary-boundary-enabler-disabler-launcher-exec-plan.md`.

## Desired End State

- A API principal de mudança de modo passa a ser `prepareModeChange(RuntimeMode)` retornando `RuntimeModeChangePlan`.
- `RuntimeExtensionInstaller`, `RuntimeExtensionModeEnabler` e `RuntimeExtensionModeDisabler` usam `apply()` em vez de carregar documento bruto na orquestração.
- A ordem de negócio fica explícita no domínio; secondary continua só com mecanismo técnico.
- Testes provam que não há regressão em fail-fast, mensagens de erro e persistência final do modo.

## Milestones

### Milestone 1 - Introduzir contrato prepare/apply no domínio e adapter

#### Goal

Adicionar o novo contrato de mudança preparada sem alterar comportamento observável.

#### Changes

1. Editar `src/main/java/com/seed4j/cli/bootstrap/domain/RuntimeModeConfigurationRepository.java` para incluir `prepareModeChange(RuntimeMode targetMode)`.
2. Criar `src/main/java/com/seed4j/cli/bootstrap/domain/RuntimeModeChangePlan.java` com `Path configPath()` e `void apply() throws IOException`.
3. Editar `src/main/java/com/seed4j/cli/bootstrap/infrastructure/secondary/FileSystemRuntimeModeConfigurationRepository.java` para produzir `RuntimeModeChangePlan` baseado em snapshot validado.
4. Preservar `readMode()` para uso no launcher no slice seguinte.

#### Validation

1. Command: `./mvnw -Dtest=RuntimeExtensionInstallerTest,RuntimeExtensionModeEnablerTest,RuntimeExtensionModeDisablerTest test`
2. Expected result: compilação e testes verdes após introdução do contrato, sem regressões comportamentais.

#### Acceptance Criteria

1. O repositório compila com o novo tipo `RuntimeModeChangePlan` disponível no domínio.
2. O adapter filesystem consegue preparar e aplicar mudança de modo sem expor `Map` cru na API principal.

### Milestone 2 - Migrar installer, enabler e disabler para o novo protocolo

#### Goal

Remover o protocolo temporal implícito dos três serviços e centralizar uso em `prepareModeChange(...).apply()`.

#### Changes

1. Editar `src/main/java/com/seed4j/cli/bootstrap/domain/RuntimeExtensionInstaller.java` para usar `prepareModeChange(RuntimeMode.EXTENSION)` e `apply()` após instalação de artefatos.
2. Editar `src/main/java/com/seed4j/cli/bootstrap/domain/RuntimeExtensionModeEnabler.java` para usar `prepareModeChange(RuntimeMode.EXTENSION)` e `apply()` após validação de runtime artifacts.
3. Editar `src/main/java/com/seed4j/cli/bootstrap/domain/RuntimeExtensionModeDisabler.java` para usar `prepareModeChange(RuntimeMode.STANDARD)` e `apply()`.
4. Remover chamadas diretas de `readConfiguration()` e `persistMode(...)` desses três serviços.

#### Validation

1. Command: `./mvnw -Dtest=RuntimeExtensionInstallerTest,RuntimeExtensionModeEnablerTest,RuntimeExtensionModeDisablerTest test`
2. Expected result: suítes verdes com ordem de operações preservada.

#### Acceptance Criteria

1. Nenhum dos três serviços manipula documento de config para persistência de modo.
2. O protocolo prepare/apply é o caminho único de mudança de modo nesses fluxos.

### Milestone 3 - Reforçar testes focados de ordem e falha

#### Goal

Garantir que a evolução da API não remove fail-fast nem altera mensagens observáveis.

#### Changes

1. Atualizar `src/test/java/com/seed4j/cli/bootstrap/domain/RuntimeExtensionInstallerTest.java` para cobrir explicitamente que `apply()` não ocorre quando instalação falha.
2. Atualizar `src/test/java/com/seed4j/cli/bootstrap/domain/RuntimeExtensionModeEnablerTest.java` para cobrir que `apply()` não ocorre quando validação de runtime artifacts falha.
3. Atualizar `src/test/java/com/seed4j/cli/bootstrap/domain/RuntimeExtensionModeDisablerTest.java` para cobrir falha de `apply()` com mesma mensagem de erro técnica atual.
4. Adicionar/ajustar teste focado do adapter em `src/test/java/...` (novo arquivo se necessário) para garantir preservação de chaves e escrita do modo alvo via `RuntimeModeChangePlan.apply()`.

#### Validation

1. Command: `./mvnw -Dtest=RuntimeExtensionInstallerTest,RuntimeExtensionModeEnablerTest,RuntimeExtensionModeDisablerTest test`
2. Expected result: testes unitários verdes com asserts explícitos de ordem e mensagens.
3. Command: `./mvnw -Dtest=ExtensionInstallCommandTest test`
4. Expected result: caminho público de `extension install` inalterado.

#### Acceptance Criteria

1. Os testes demonstram que falhas antes de `apply()` não persistem `mode` indevidamente.
2. Mensagens de erro permanecem compatíveis com as suítes atuais.

### Milestone 4 - Consolidação e validação ampla

#### Goal

Fechar o slice com validação ampla e atualização documental para handoff seguro.

#### Changes

1. Atualizar este ExecPlan com progresso final, decisões, riscos atualizados e lições aprendidas.
2. Atualizar referências cruzadas no plano de runtime-mode de 2026-05-26 com link para este plano quando concluído.
3. Remover TODOs temporários introduzidos durante a discussão para manter base limpa.

#### Validation

1. Command: `./mvnw clean verify`
2. Expected result: validação ampla do repositório verde.
3. Command: `npm run prettier:check`
4. Expected result: sem divergências de formatação.

#### Acceptance Criteria

1. Plano atualizado como documento vivo e reutilizável por outro engenheiro.
2. Suítes focadas e validação ampla sem regressões atribuíveis a este refactor.

## Progress

- [x] Milestone 1 started
- [x] Milestone 1 completed
- [ ] Milestone 2 started
- [ ] Milestone 2 completed
- [ ] Milestone 3 started
- [ ] Milestone 3 completed
- [ ] Milestone 4 started
- [ ] Milestone 4 completed

### TDD Cycle Log

- Cycle 1 | Introduzir `prepareModeChange(...).apply()` no adapter com preservação de chaves ao persistir modo | expected failure: erro de compilação por ausência de `RuntimeModeChangePlan` e `prepareModeChange(...)` | 🔴 red result: falhou compilação em `FileSystemRuntimeModeConfigurationRepositoryTest` com símbolos ausentes | 🌱 green change: criado `RuntimeModeChangePlan`, adicionada API `prepareModeChange(...)` na porta e implementação no `FileSystemRuntimeModeConfigurationRepository` baseada em snapshot | suite result: `./mvnw -Dtest=FileSystemRuntimeModeConfigurationRepositoryTest,RuntimeExtensionInstallerTest,RuntimeExtensionModeEnablerTest,RuntimeExtensionModeDisablerTest test` verde (23 testes) | vertical checkpoint (when due): n/a | 🌀 refactor: sem simplificação necessária no ciclo
- Cycle 2 | Garantir fail-fast no `prepareModeChange(...)` com YAML inválido | expected failure: falha do novo teste se preparo não validasse configuração | 🔴 red result: teste novo já passou, confirmando fail-fast no preparo | 🌱 green change: sem mudança de produção (comportamento já atendido pelo código do ciclo 1) | suite result: suíte relevante verde (23 testes) | vertical checkpoint (when due): `./mvnw -Dtest=ExtensionInstallCommandTest test` verde (6 testes) | 🌀 refactor: sem simplificação necessária no ciclo

## Decisions

- Decision: Adotar nomenclatura da opção 1: `prepareModeChange(...)` + `RuntimeModeChangePlan.apply()`.
  Rationale: comunica protocolo em duas fases sem semântica transacional inflada (`commit`).
  Date/Author: 2026-05-27 / Renan + Codex

- Decision: A regra de ordenação de passos fica no domínio; detalhes técnicos de I/O/YAML ficam no secondary.
  Rationale: preservar fronteira arquitetural e evitar política de negócio dentro do adapter.
  Date/Author: 2026-05-27 / Renan + Codex

- Decision: Manter testes focados como gate obrigatório deste refactor.
  Rationale: mudança de protocolo pode regredir fail-fast e ordem sem falhar em testes superficiais.
  Date/Author: 2026-05-27 / Renan + Codex

- Decision: Não incluir locking trans-processo neste slice.
  Rationale: fora do escopo de mudança mínima; foco em explicitar protocolo e preservar comportamento.
  Date/Author: 2026-05-27 / Renan + Codex

- Decision: Preservar `readMode()` na porta atual para compatibilidade com fluxo de launcher no plano incremental.
  Rationale: evita churn de assinatura fora do escopo imediato.
  Date/Author: 2026-05-27 / Renan + Codex

- Decision: Introduzir `prepareModeChange(RuntimeMode)` como método default da porta nesta milestone.
  Rationale: manter compatibilidade com doubles de teste e fluxos ainda não migrados, enquanto o adapter filesystem já expõe o protocolo prepare/apply.
  Date/Author: 2026-05-27 / Renan + Codex

## Risks and Mitigations

- Risk: mudar nomes de porta quebrar doubles de teste em vários arquivos.
  Mitigation: migrar tests na mesma milestone da mudança de assinatura e validar suíte focada completa.

- Risk: regressão de fail-fast no installer ao aplicar modo antes/depois do ponto correto.
  Mitigation: asserts explícitos de ordem no teste do installer e cenário de falha de instalação sem `apply()`.

- Risk: regressão de mensagens de erro técnicas em enable/disable.
  Mitigation: manter asserts de mensagem e causa (`IOException`/`YAMLException`) nas suítes existentes.

- Risk: ambiguidade de semântica em `apply()` (parecer idempotente sem ser).
  Mitigation: documentar no tipo de domínio que o plano representa snapshot preparado para aplicação única por fluxo.

## Validation Strategy

1. Rodar testes focados de domínio: installer, enabler e disabler.
2. Rodar teste de caminho público do comando: `ExtensionInstallCommandTest`.
3. Rodar validação ampla: `./mvnw clean verify`.
4. Rodar formatação: `npm run prettier:check`.

## Rollout and Recovery

Rollout:
1. Entregar como refactor interno sem mudança de contrato CLI.
2. Revisar diff final para garantir que somente protocolo interno mudou.

Recovery:
1. Reverter commits deste plano caso apareça regressão de ordem/fail-fast.
2. Se regressão ficar isolada em enable/disable, manter installer no novo protocolo e rollback seletivo dos serviços afetados.

## Lessons Learned

- Nome de API impacta expectativa de consistência; `apply` foi preferido por ser forte sem insinuar transação completa.
- Acoplamento temporal não some; ele deve ser explicitado em protocolo de domínio para reduzir erro de uso.
- Testes focados de ordem são essenciais quando o objetivo é segurança de orquestração e não feature nova.
