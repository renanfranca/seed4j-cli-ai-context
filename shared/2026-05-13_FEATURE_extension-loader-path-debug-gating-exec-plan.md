# Gate de diagnostico `loader.path` por `--debug` em extension mode

This ExecPlan is a living document. Keep `Progress`, `Decisions`, `Risks`, and `Lessons Learned` up to date as work advances.

Safety boundary: This task is limited to authorized, defensive maintenance of this repository. Do not provide offensive guidance or policy-bypassing instructions.

## Purpose / Big Picture

No extension mode, o bootstrap passou a registrar logs DEBUG sobre bibliotecas adicionadas ao `loader.path`. O objetivo deste plano e garantir que esse diagnostico seja observado somente quando o usuario executar o CLI com `--debug`, mantendo a saida padrao limpa. O comportamento observavel esperado e: sem `--debug` nao ha ruido adicional de diagnostico; com `--debug` os logs de decisao de `loader.path` ficam disponiveis para troubleshooting.

## Scope

In scope: ajustar o status do milestone existente para pendente, implementar o gate de diagnostico por `--debug` no caminho publico do launcher, e validar com testes automatizados.

Out of scope: alterar estrategia de selecao de bibliotecas ausentes, alterar contrato do gerador `seed4j-extension`, ou expandir escopo para outros tipos de logs fora da decisao de `loader.path`.

## Definitions

`extension mode`: execucao do CLI com runtime externo ativo (`seed4j.cli.runtime.mode=extension`).

`diagnostico loader.path`: mensagens de DEBUG que listam quais jars de `BOOT-INF/lib` da extensao foram efetivamente adicionados ao `loader.path`.

`gate por --debug`: condicao em que o diagnostico so deve ser emitido quando a chamada de CLI incluir a flag `--debug`.

## Existing Context

O comportamento atual de diagnostico esta em `src/main/java/com/seed4j/cli/bootstrap/domain/RuntimeExtensionLoaderPathResolver.java`, com `LOGGER.debug(...)` no metodo `resolve`.

No fluxo de extension mode, o launcher seta `logging.level.root=ERROR` em `src/main/java/com/seed4j/cli/bootstrap/domain/Seed4JCliLauncher.java`, o que pode ocultar logs DEBUG mesmo quando o usuario pede `--debug`.

O plano principal do feature (`2026-04-29_FEATURE_extension-mode-loader-path-filtered-overlay-exec-plan.md`) havia marcado esse item como concluido, mas a exigencia atual e tratá-lo como pendente ate comprovacao no caminho publico com `--debug`.

## Desired End State

O plano principal do feature registra esse item como pendente.

O runtime em extension mode emite diagnostico de `loader.path` apenas quando a execucao incluir `--debug`.

A validacao automatizada prova os dois caminhos observaveis: sem `--debug` (sem diagnostico) e com `--debug` (com diagnostico).

## Milestones

### Milestone 1 - Reclassificar pendencia no plano principal

Goal: corrigir o status no plano principal para nao reportar como concluido antes da validacao do gate por `--debug`.

File-level edits:
- `_temporary/ai_agent/seed4j-cli-ai-context/shared/2026-04-29_FEATURE_extension-mode-loader-path-filtered-overlay-exec-plan.md`: mudar item de logs DEBUG para pendente no bloco `Changes`.
- `_temporary/ai_agent/seed4j-cli-ai-context/shared/2026-04-29_FEATURE_extension-mode-loader-path-filtered-overlay-exec-plan.md`: mudar `Additional command/result` de logs DEBUG para `Pending`.
- `_temporary/ai_agent/seed4j-cli-ai-context/shared/2026-04-29_FEATURE_extension-mode-loader-path-filtered-overlay-exec-plan.md`: atualizar risco para refletir gate por `--debug` ainda aberto.

Validation commands:
- `nl -ba _temporary/ai_agent/seed4j-cli-ai-context/shared/2026-04-29_FEATURE_extension-mode-loader-path-filtered-overlay-exec-plan.md | sed -n '156,210p'`
- `nl -ba _temporary/ai_agent/seed4j-cli-ai-context/shared/2026-04-29_FEATURE_extension-mode-loader-path-filtered-overlay-exec-plan.md | sed -n '340,352p'`

Acceptance criteria:
- O item de logs DEBUG nao aparece mais como concluido no milestone 4.
- O risco pendente menciona explicitamente o gate por `--debug`.

### Milestone 2 - Implementar gate de diagnostico no caminho publico

Goal: garantir que logs de diagnostico de `loader.path` sejam observaveis apenas quando `--debug` for informado.

File-level edits:
- `src/main/java/com/seed4j/cli/bootstrap/domain/Seed4JCliLauncher.java`: detectar `--debug` nos argumentos recebidos e propagar configuracao de logging apropriada para o child process.
- `src/main/java/com/seed4j/cli/bootstrap/domain/RuntimeExtensionLoaderPathResolver.java`: manter mensagens de DEBUG focadas na decisao de libs adicionadas.
- `src/test/java/com/seed4j/cli/bootstrap/domain/Seed4JCliLauncherTest.java`: cobrir propagacao de configuracao no request do child process com e sem `--debug`.
- `src/test/java/com/seed4j/cli/bootstrap/domain/RuntimeExtensionLoaderPathResolverTest.java`: cobrir comportamento observavel do diagnostico no resolver conforme configuracao recebida no runtime.

Validation commands:
- `./mvnw -Dtest=Seed4JCliLauncherTest,RuntimeExtensionLoaderPathResolverTest test`

Acceptance criteria:
- Sem `--debug`, logs de diagnostico de `loader.path` nao aparecem.
- Com `--debug`, logs de diagnostico de `loader.path` aparecem com os nomes das libs adicionadas.

### Milestone 3 - Consolidar validacao e sincronizar plano principal

Goal: fechar a pendencia no plano principal somente apos evidencia automatizada do comportamento.

File-level edits:
- `_temporary/ai_agent/seed4j-cli-ai-context/shared/2026-04-29_FEATURE_extension-mode-loader-path-filtered-overlay-exec-plan.md`: promover item de pending para concluido apos validacao real.
- `_temporary/ai_agent/seed4j-cli-ai-context/shared/2026-05-13_FEATURE_extension-loader-path-debug-gating-exec-plan.md`: atualizar progresso, decisoes e learnings finais.

Validation commands:
- `./mvnw -Dtest=Seed4JCliLauncherTest,RuntimeExtensionLoaderPathResolverTest test`
- `./mvnw -Dtest=RuntimeSelectionTest,Seed4JCliLauncherTest,ExtensionRuntimeBootstrapInProcessTest,RuntimeExtensionMissingLibrariesSelectorTest,RuntimeExtensionLoaderPathResolverTest test`

Acceptance criteria:
- O plano principal so marca concluido apos comandos verdes.
- O comportamento final fica documentado com comando e resultado observavel.

## Progress

- [x] Milestone 1 started
- [x] Milestone 1 completed
- [ ] Milestone 2 started
- [ ] Milestone 2 completed
- [ ] Milestone 3 started
- [ ] Milestone 3 completed

## Decisions

- Decision: O status de logs DEBUG no plano principal deve permanecer pendente ate validacao do gate por `--debug`.
  Rationale: evita relatar conclusao antes da evidencia no caminho publico.
  Date/Author: 2026-05-13 / User + Codex

- Decision: A implementacao deve priorizar observabilidade defensiva sem poluir a saida padrao.
  Rationale: diagnostico detalhado e util em troubleshooting, mas nao deve impactar UX default.
  Date/Author: 2026-05-13 / User + Codex

## Risks and Mitigations

- Risk: `--debug` pode nao ser propagado de forma consistente para o child process em extension mode.
  Mitigation: cobrir `Seed4JCliLauncherTest` no request final de system properties.

- Risk: logs DEBUG podem continuar ocultos por `logging.level.root=ERROR` mesmo com `--debug`.
  Mitigation: aplicar configuracao direcionada por logger de classe e validar em teste.

- Risk: marcar como concluido sem validacao de caminho publico reabre regressao documental.
  Mitigation: manter item pendente no plano principal ate milestone 3.

## Validation Strategy

1. Validar primeiro testes focados (`Seed4JCliLauncherTest`, `RuntimeExtensionLoaderPathResolverTest`).
2. Executar suite ampliada do milestone 4 para detectar regressao lateral.
3. Atualizar o plano principal somente apos evidencia verde.

## Rollout and Recovery

Rollout: promover o item para concluido no plano principal somente apos comandos de validacao passarem.

Recovery: se o gate quebrar logs ou comportamento de launch, reverter os commits dos milestones 2 e 3, manter item pendente no plano principal e reabrir investigacao.

## Lessons Learned

- Ajustes de observabilidade precisam de validacao no caminho publico, nao apenas em teste local de logger.
- Status de ExecPlan deve refletir evidencia executada, nao intencao de implementacao.
