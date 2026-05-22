# BUG: Extension-only slug visível no `list` em modo standard

- Status: resolved
- Reported on: 2026-03-23
- Resolved on: 2026-03-25
- Reported by: AI agent execution log (`Seed4JCommandsFactoryTest`)
- Severity: medium
- Resolution Plan: `_temporary/ai_agent/seed4j-cli-ai-context/shared/BUG_STANDARD_MODE_EXTENSION_LEAK_EXEC_PLAN.md`

## Summary

O slug `runtime-extension-list-only` apareceu na saída de `seed4j list` em contexto de execução tratado como standard. Esse comportamento viola o isolamento esperado entre catálogo standard e catálogo de extensão.

## Expected behavior

No modo standard, `seed4j list` não deve exibir slugs exclusivos de extensão.

## Actual behavior

Durante execução de testes de integração do comando (`Seed4JCommandsFactoryTest`), a saída de `list` incluiu `runtime-extension-list-only`.

## How to reproduce

1. Executar: `./mvnw -Dtest=Seed4JCommandsFactoryTest test`
2. Inspecionar a saída capturada para o comando `list`
3. Verificar presença de `runtime-extension-list-only`

## Impact

- Reduz confiança no isolamento de runtime mode.
- Torna asserts de catálogo standard potencialmente não determinísticos.
- Pode mascarar regressões reais ao misturar catálogo base e catálogo de extensão.

## Suspected area

- Seleção de runtime e carregamento de extensão na montagem de comandos (`CliFixture` / `RuntimeSelection` / bootstrap de módulos).

## Suggested fix direction

1. Garantir que a execução standard use apenas catálogo base por default.
2. Carregar módulos de extensão apenas quando extension mode estiver explicitamente ativo.
3. Adicionar teste de proteção: standard mode deve falhar se `runtime-extension-list-only` aparecer no `list`.

## Resolution Summary

- Teste de protecao adicionado em `Seed4JCommandsFactoryTest` para garantir que `list` em standard mode nao contenha `runtime-extension-list-only`.
- Causa raiz corrigida em `RuntimeExtensionListOnlyModuleConfiguration` com ativacao condicional por `seed4j.cli.runtime.mode=extension`.
- Evidencias de validacao (todas verdes em 2026-03-25):
  - `./mvnw -Dtest=Seed4JCommandsFactoryTest test` (red antes da correcao, green depois)
  - `./mvnw -Dtest=CurrentProcessRuntimeSelectionProviderTest,SystemPropertyRuntimeSelectionProviderTest test`
  - `./mvnw -Dit.test=ExtensionRuntimeBootstrapListPackagedJarIT failsafe:integration-test failsafe:verify`
  - `./mvnw clean verify`
  - `npm run prettier:check`
