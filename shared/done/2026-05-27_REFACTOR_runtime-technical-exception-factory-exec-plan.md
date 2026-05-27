# Padronizar Tratamento de Exceções Técnicas de Runtime (I/O + YAML) com Factory Central

Este ExecPlan é um documento vivo. Mantenha `Progress`, `Decisions`, `Risks` e `Lessons Learned` atualizados durante a execução.

Safety boundary: This task is limited to authorized, defensive maintenance of this repository. Do not provide offensive guidance or policy-bypassing instructions.

## Purpose / Big Picture

A CLI hoje falha em vários pontos de runtime com mensagens inconsistentes: alguns fluxos perdem a causa, outros concatenam `getMessage()` localmente e outros escondem detalhes úteis.
Vamos padronizar o tratamento técnico de I/O e YAML para que toda falha funcional preserve `cause` e exponha detalhe técnico de forma consistente no texto retornado ao usuário.
A referência adotada é o padrão do `seed4j/module`: centralizar construção de exceção técnica e evitar formatação de erro espalhada.

## Scope

Em escopo:
- Erros funcionais de I/O/YAML no bootstrap/runtime que hoje viram `InvalidRuntimeConfigurationException`.
- Centralização da composição de mensagem técnica em um único ponto.
- Ajustes de testes para refletir detalhe dinâmico sem fragilidade.

Fora de escopo:
- Mudança de contrato funcional dos comandos.
- Alteração de exceções puramente de validação de domínio (sem I/O/YAML).
- Mudanças em cleanup best-effort que são intencionalmente silenciosas.

## Definitions

- **Exceção técnica**: falha de infraestrutura/parsing (I/O, YAML) encapsulada para diagnóstico.
- **Mensagem base**: texto estável e orientado ao usuário (ex.: “Could not read ...”).
- **Detalhe técnico**: `cause.getMessage()` anexado ao final como `Details: ...` quando disponível.
- **Factory central**: método estático responsável por montar a exceção técnica com formatação única.

## Existing Context

- `InvalidRuntimeConfigurationException` aceita apenas `String` e não impõe padrão único de mensagem/cause.
- Há mistura de estilos: mensagem genérica sem causa (`mode enabler/disabler`), concatenação manual (`installer`, `overlay`, `loader`) e perdas de causa em alguns fluxos.
- A CLI imprime majoritariamente `exception.getMessage()`, então a mensagem precisa carregar diagnóstico suficiente.
- No `seed4j/module`, o padrão predominante é `technicalError(message, cause)` centralizado e com `cause` preservada.

## Desired End State

- Toda falha funcional de I/O/YAML no runtime é lançada via factory técnica central de `InvalidRuntimeConfigurationException`.
- A composição da mensagem técnica (`Details:` + fallback para cause sem mensagem) fica em um único lugar.
- Nenhum ponto de conversão funcional de I/O/YAML concatena `getMessage()` manualmente.
- Testes validam mensagem base + detalhe quando aplicável + `cause` preservada.

## Milestones

### Milestone 1 - Introduzir contrato técnico central da exceção

#### Goal

Criar a API mínima para padronizar mensagens técnicas e preservar causa sem duplicação de lógica.

#### Changes

- [x] Atualizar `InvalidRuntimeConfigurationException` com:
  - construtor `(String message, Throwable cause)`;
  - factory estática (ex.: `technicalError(String baseMessage, Throwable cause)`) para montar mensagem final;
  - helper interno para anexar `Details:` só quando houver conteúdo útil.
- [x] Definir fallback para `cause.getMessage()` vazio/nulo (usar `cause.getClass().getSimpleName()` como detalhe mínimo).

#### Validation

- [x] Command: `./mvnw -Dtest=RuntimeExtensionModeEnablerTest,RuntimeExtensionModeDisablerTest test`
- [x] Expected result: testes compilam com novo contrato de exceção e sem regressão funcional.

#### Acceptance Criteria

- [x] Existe um único caminho recomendado para exceções técnicas.
- [x] Mensagem técnica nunca termina com `Details: null` ou vazio.
- [x] `cause` sempre preservada no objeto de exceção.

### Milestone 2 - Migrar fluxos de config/install para a factory central

#### Goal

Eliminar mascaramento de causa e concatenação local nos fluxos de maior impacto ao usuário.

#### Changes

- [x] Migrar `RuntimeModeConfigReader` para usar factory técnica em `IOException` e `YAMLException` (separando de validações semânticas).
- [x] Migrar `RuntimeExtensionModeEnabler`, `RuntimeExtensionModeDisabler` e `RuntimeExtensionInstaller` para factory técnica.
- [x] Manter mensagens base atuais para compatibilidade de entendimento.

#### Validation

- [x] Command: `./mvnw -Dtest=RuntimeExtensionModeEnablerTest,RuntimeExtensionModeDisablerTest,RuntimeExtensionInstallerTest,ExtensionInstallCommandTest test`
- [x] Expected result: mensagens com base estável + `Details:`; erros continuam mapeando para saída não-zero.

#### Acceptance Criteria

- [x] Casos de persistência falha exibem `Details: cannot persist` (ou equivalente).
- [x] Casos de YAML inválido exibem detalhe do parser.
- [x] Nenhum desses fluxos usa `+ ioException.getMessage()` manual.

### Milestone 3 - Migrar runtime artifacts/process e endurecer testes

#### Goal

Concluir padronização nos demais pontos técnicos de runtime e estabilizar contrato de teste.

#### Changes

- [x] Migrar para factory técnica os pontos de I/O/YAML em: `RuntimeMetadata`, `RuntimeExtensionStartClassResolver`, `RuntimeExtensionJarLayoutValidator`, `RuntimeExtensionOverlayCache`, `RuntimeExtensionLoaderPathResolver`, `RuntimeExtensionCacheIdentityResolver`, `Seed4JCliApp` (child process I/O).
- [x] Ajustar testes para:
  - usar `hasMessageContaining` na parte estável;
  - validar presença de detalhe técnico quando determinístico;
  - validar `hasCauseInstanceOf(...)` nos cenários com causa controlada.
- [x] Manter exceções de validação de domínio (sem I/O/YAML) com assert estrito de mensagem quando já existente.

#### Validation

- [x] Command: `./mvnw -Dtest=RuntimeSelectionTest,RuntimeExtensionStartClassResolverTest,RuntimeExtensionOverlayCacheTest,RuntimeExtensionLoaderPathResolverTest,Seed4JCliAppTest test`
- [x] Expected result: suíte focada verde com contrato consistente.
- [x] Command: `./mvnw clean verify`
- [x] Expected result: validação completa verde (tests + cobertura + quality gates).

#### Acceptance Criteria

- [x] Não há regressão comportamental de comandos.
- [x] Contrato técnico de erro está consistente em todos os pontos funcionais de I/O/YAML no runtime.
- [x] Build completa passa.

## Progress

- [x] Milestone 1 started
- [x] Milestone 1 completed
- [x] Milestone 2 started
- [x] Milestone 2 completed
- [x] Milestone 3 started
- [x] Milestone 3 completed

## Decisions

- Decision: adotar factory central de exceção técnica em `InvalidRuntimeConfigurationException`.
  Rationale: reduzir inconsistência de formatação e alinhamento com padrão observado no `seed4j/module`.
  Date/Author: 2026-05-27 / Codex + Renan

- Decision: manter `Details:` como sufixo padrão para detalhe técnico.
  Rationale: separa mensagem base (legível) de detalhe de diagnóstico (acionável).
  Date/Author: 2026-05-27 / Codex + Renan

- Decision: preservar exceções de validação semântica sem forçar detalhe técnico.
  Rationale: evitar poluir erros de regra de domínio com ruído de infraestrutura.
  Date/Author: 2026-05-27 / Codex

- Decision: não alterar catches de cleanup best-effort.
  Rationale: cleanup falho não deve transformar sucesso funcional em erro.
  Date/Author: 2026-05-27 / Codex

- Decision: manter mapeamento defensivo de `RuntimeException` inesperada em `RuntimeMetadata.read(...)` para erro funcional estável.
  Rationale: evita escapar exceções técnicas inesperadas com stacktrace bruto para o usuário.
  Date/Author: 2026-05-27 / Codex

## Risks and Mitigations

- Risk: detalhes de parser YAML variam entre versões/ambientes e quebram asserts exatos.
  Mitigation: asserts por trecho estável + validação de tipo da `cause`.

- Risk: excesso de informação técnica na saída de CLI.
  Mitigation: manter mensagem base curta; detalhe como sufixo opcional.

- Risk: implementação parcial deixar padrões mistos.
  Mitigation: checklist por subsistema e busca final para eliminar concatenação manual de `getMessage()`.

## Validation Strategy

1. Executar testes focados por milestone.
2. Verificar contrato de mensagens/cause após cada migração de bloco.
3. Executar `./mvnw clean verify` ao final.
4. Confirmar aceites de cada milestone antes de finalizar.

## Rollout and Recovery

- Rollout: merge direto; mudança é de observabilidade/diagnóstico sem alterar fluxo nominal.
- Recovery: rollback do último commit/milestone se qualquer regressão de mensagem ou comportamento for detectada.
- Pós-rollout: observar feedback de clareza de erro em execução real de comandos de extension/runtime.

## Lessons Learned

- Registro inicial: padrão centralizado evita erros sutis de concatenação e reduz drift de mensagens entre classes.
- SnakeYAML pode encapsular causas de I/O como `YAMLException` em cenários de leitura; asserts devem focar no contrato estável (`Details:` + texto) e no tipo de causa mais próximo observado.
- Gatilhos de cobertura por classe exigiram testes de ramos pouco usuais (detalhe técnico vazio em classe anônima e falha de abertura de metadata) para manter gates estritos sem relaxar o contrato.
