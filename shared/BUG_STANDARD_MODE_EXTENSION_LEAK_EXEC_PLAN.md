# Corrigir vazamento de modulo de extensao no `seed4j list` em modo standard

Este ExecPlan e um documento vivo. Atualize `Progress`, `Decisions`, `Risks` e `Lessons Learned` durante a execucao.

## Purpose / Big Picture

Hoje existe um bug reportado em `_temporary/ai_agent/seed4j-cli-ai-context/shared/BUG_STANDARD_MODE_EXTENSION_LEAK.md`: o slug `runtime-extension-list-only` aparece no `seed4j list` mesmo em contexto standard. Isso quebra o isolamento entre catalogo base e catalogo de extensao e reduz a confianca nos testes de comando. O objetivo desta entrega e garantir comportamento observavel e deterministico: em standard mode o slug de extensao nao aparece; em extension mode ele aparece de forma aditiva e sem duplicidade.

## Scope

Escopo:

- Reproduzir e fixar o bug com um teste de protecao explicito para standard mode.
- Corrigir a causa de vazamento de beans/modulos de extensao no contexto Spring usado pelos testes in-process.
- Garantir que extension mode continue funcionando (slug aparece somente quando runtime de extensao estiver ativo).
- Validar com testes focados e `./mvnw clean verify`.

Fora de escopo:

- Refatoracao ampla de bootstrap/runtime que nao esteja ligada ao vazamento.
- Mudanca de UX do comando `list` (colunas, layout, ordenacao etc.).
- Alteracoes no contrato externo de configuracao (`~/.config/seed4j-cli.yml`) alem do necessario para garantir o isolamento.

## Definitions

- `standard mode`: runtime padrao da CLI, sem carregamento de `extension.jar`.
- `extension mode`: runtime que adiciona modulos de extensao carregados via `loader.path`.
- `extension-only slug`: slug disponivel apenas em extensao (`runtime-extension-list-only`).
- `component scan leakage`: quando classes de fixture/teste anotadas como componentes entram no contexto Spring de cenarios que nao deveriam conhece-las.
- `in-process command tests`: testes que exercitam comandos via `CommandLine` no mesmo processo JVM de teste (ex.: `Seed4JCommandsFactoryTest`).

## Existing Context

- Bug de referencia: `_temporary/ai_agent/seed4j-cli-ai-context/shared/BUG_STANDARD_MODE_EXTENSION_LEAK.md` (status `open`, reportado em 2026-03-23).
- O teste que reportou o sintoma foi `src/test/java/com/seed4j/cli/command/infrastructure/primary/Seed4JCommandsFactoryTest.java`.
- O bootstrap de runtime define `seed4j.cli.runtime.mode` e `loader.path` apenas no caminho de child process: `src/main/java/com/seed4j/cli/bootstrap/domain/Seed4JCliLauncher.java`.
- O provider de runtime em comandos le propriedades de sistema e assume `standard` por default: `src/main/java/com/seed4j/cli/command/infrastructure/primary/SystemPropertyRuntimeSelectionProvider.java`.
- O app usa `@SpringBootApplication(scanBasePackageClasses = { Seed4JApp.class, Seed4JCliApp.class })`: `src/main/java/com/seed4j/cli/Seed4JCliApp.java`.
- Existem classes de fixture de extensao em `src/test/java/com/seed4j/cli/bootstrap/domain/runtimeextension/list/*` anotadas com `@Configuration`/`@Service`, incluindo um bean `Seed4JModuleResource` para `runtime-extension-list-only`.
- O comportamento esperado ja esta coberto no teste empacotado: `src/test/java/com/seed4j/cli/bootstrap/domain/ExtensionRuntimeBootstrapListPackagedJarIT.java` (standard nao lista slug de extensao; extension lista).

## Desired End State

- `seed4j list` em contexto standard nao exibe `runtime-extension-list-only`.
- `seed4j list` em extension mode continua exibindo `runtime-extension-list-only` sem remover modulos standard e sem duplicidade.
- Testes in-process e empacotados cobrem explicitamente a fronteira standard vs extension.
- Causa raiz fica documentada em `Decisions`/`Lessons Learned` para evitar regressao futura.

## Milestones

### Milestone 1 - Reproducao deterministica e teste de protecao

#### Goal

Transformar o bug em um teste claro que falha quando houver vazamento no modo standard.

#### Changes

- [x] Editar `src/test/java/com/seed4j/cli/command/infrastructure/primary/Seed4JCommandsFactoryTest.java` e adicionar/ajustar teste no bloco `list` para afirmar explicitamente que saida standard nao contem `runtime-extension-list-only`.
- [x] Garantir que o cenario use runtime standard (sem propriedades de extensao forjadas).
- [x] Manter assercoes inline no corpo do teste (Given/When/Then separado por linha em branco, conforme guia do repositorio).

#### Validation

- [x] Command: `./mvnw -Dtest=Seed4JCommandsFactoryTest test`
- [x] Expected result: teste falha antes da correção (red) se o vazamento estiver presente; apos a correção, passa e protege contra regressao.

#### Acceptance Criteria

- [x] Existe teste especifico e legivel que captura o vazamento no modo standard.
- [x] A falha aponta diretamente para o slug de extensao indevido.

### Milestone 2 - Isolar carregamento da fixture de extensao

#### Goal

Remover a fonte do vazamento sem quebrar o comportamento esperado de extension mode.

#### Changes

- [x] Ajustar `src/test/java/com/seed4j/cli/bootstrap/domain/runtimeextension/list/RuntimeExtensionListOnlyModuleConfiguration.java` para registrar o modulo de fixture somente sob condicao explicita de runtime de extensao (por propriedade de sistema usada no bootstrap).
- [x] Ajustar classes auxiliares da fixture no mesmo pacote apenas se necessario para manter wiring consistente.
- [x] Evitar solução que filtre catalogo no comando `list` para mascarar vazamento de contexto; a prioridade e corrigir a origem do bean indevido.

#### Validation

- [x] Command: `./mvnw -Dtest=Seed4JCommandsFactoryTest test`
- [x] Expected result: `list` em standard mode nao mostra `runtime-extension-list-only`.
- [x] Command: `./mvnw -Dtest=CurrentProcessRuntimeSelectionProviderTest,SystemPropertyRuntimeSelectionProviderTest test`
- [x] Expected result: contrato de leitura de runtime properties permanece estavel.

#### Acceptance Criteria

- [x] Bean/modulo de fixture nao entra no contexto standard por default.
- [x] A correção atua na causa raiz (registro indevido de modulo), nao apenas no sintoma.

### Milestone 3 - Proteger comportamento de extension mode

#### Goal

Confirmar que extension mode continua aditivo apos isolar o modo standard.

#### Changes

- [x] Revisar (e, se necessario, reforcar) cobertura em `src/test/java/com/seed4j/cli/bootstrap/domain/ExtensionRuntimeBootstrapListPackagedJarIT.java` para manter os tres cenarios: standard sem slug de extensao, extension com slug, e delta aditivo sem remocoes.
- [x] Ajustar fixture de extensao (`src/test/java/com/seed4j/cli/bootstrap/domain/ExtensionRuntimeFixture.java`) apenas se a nova condicao de ativacao exigir dados extras.

#### Validation

- [x] Command: `./mvnw -Dit.test=ExtensionRuntimeBootstrapListPackagedJarIT failsafe:integration-test failsafe:verify`
- [x] Expected result: todos os cenarios empacotados passam; extension-only slug aparece somente em extension mode.

#### Acceptance Criteria

- [x] Extension mode continua adicionando `runtime-extension-list-only` sem duplicidade.
- [x] Standard catalog e preservado integralmente.

### Milestone 4 - Validacao completa e fechamento

#### Goal

Fechar com validacao de repositorio e rastreabilidade do bug.

#### Changes

- [x] Atualizar `_temporary/ai_agent/seed4j-cli-ai-context/shared/BUG_STANDARD_MODE_EXTENSION_LEAK.md` com status final e referencia para este ExecPlan quando a implementacao terminar.
- [x] Registrar decisoes finais, riscos residuais e aprendizados neste arquivo.

#### Validation

- [x] Command: `./mvnw clean verify`
- [x] Expected result: build completo verde, sem regressao de cobertura/checkstyle/tests.
- [x] Command: `npm run prettier:check`
- [x] Expected result: sem divergencia de formatacao nos arquivos alterados.

#### Acceptance Criteria

- [x] Bug marcado como resolvido com evidencia de teste.
- [x] Existe trilha clara de verificacao local para reproduzir e validar a correção.

## Progress

- [x] Milestone 1 started
- [x] Milestone 1 completed
- [x] Milestone 2 started
- [x] Milestone 2 completed
- [x] Milestone 3 started
- [x] Milestone 3 completed
- [x] Milestone 4 started
- [x] Milestone 4 completed

## Decisions

- Decision: priorizar isolamento na origem (registro/ativacao de bean de extensao) em vez de filtrar saida do comando `list`.
  Rationale: filtro na saida esconderia vazamento arquitetural e poderia contaminar outros comandos (como `apply`).
  Date/Author: 2026-03-25 / Codex

- Decision: ativar `RuntimeExtensionListOnlyModuleConfiguration` apenas quando `seed4j.cli.runtime.mode=extension`.
  Rationale: a propriedade ja e parte do contrato do bootstrap em child process e elimina o bean de fixture no contexto standard sem alterar o contrato do comando `list`.
  Date/Author: 2026-03-25 / Codex

- Decision: manter validacao empacotada como criterio obrigatorio de aceite para extension mode.
  Rationale: esse caminho representa o fluxo real de runtime (child process + `loader.path`) e reduz falso positivo de testes in-process.
  Date/Author: 2026-03-25 / Codex

## Risks and Mitigations

- Risk: Condicao de ativacao escolhida para fixture pode bloquear indevidamente o carregamento em extension mode.
  Mitigation: validar com `ExtensionRuntimeBootstrapListPackagedJarIT` e checar presenca explicita do slug de extensao.

- Risk: Correcao ficar acoplada a detalhes de teste e nao refletir o comportamento real de runtime.
  Mitigation: exigir validacao em jar empacotado alem de testes in-process.

- Risk: Regressao silenciosa em outros comandos que usam o mesmo catalogo de modulos.
  Mitigation: executar `./mvnw clean verify` completo para cobrir fluxos adjacentes.

- Risk: Teste tornar-se nao deterministico por poluicao de system properties entre cenarios.
  Mitigation: manter captura/restauracao explicita de propriedades quando necessario e evitar dependencia em estado global residual.

## Validation Strategy

1. Reproduzir o bug no teste de comando focado (`Seed4JCommandsFactoryTest`).
2. Aplicar correcao de isolamento de fixture/bean de extensao.
3. Reexecutar testes focados de comando e runtime selection providers.
4. Rodar IT empacotado de extension list para validar fronteira standard vs extension.
5. Rodar `./mvnw clean verify` e `npm run prettier:check`.

## Validation Evidence

- `./mvnw -Dtest=Seed4JCommandsFactoryTest test`
  - Red observado apos adicionar teste: falha por presenca de `runtime-extension-list-only` em standard mode.
  - Green observado apos condicao no bean de fixture: classe passou com catalogo de 168 modulos e sem slug de extensao.
- `./mvnw -Dtest=CurrentProcessRuntimeSelectionProviderTest,SystemPropertyRuntimeSelectionProviderTest test`
  - Passou (2 testes), mantendo contrato de leitura de propriedades de runtime.
- `./mvnw -Dit.test=ExtensionRuntimeBootstrapListPackagedJarIT failsafe:integration-test failsafe:verify`
  - Passou (3 testes), preservando cenarios standard/extension/delta aditivo.
- `./mvnw clean verify`
  - Passou (unitarios + integracao + cobertura + checkstyle).
- `npm run prettier:check`
  - Passou sem divergencia de formatacao.

## Rollout and Recovery

Rollout:

1. Entregar correcao em commit focado (`fix(bootstrap)` ou `fix(command)` conforme arquivos tocados).
2. Anexar evidencias dos comandos de validacao no PR.

Recovery:

1. Se extension mode deixar de carregar modulo, reverter condicao de ativacao da fixture e reavaliar criterio (ex.: propriedade de runtime mais adequada).
2. Se standard continuar vazando slug, pausar merge e reabrir investigacao de component scan com teste minimo reproduzivel.

## Lessons Learned

- Fixtures de teste anotadas com `@Configuration` dentro de pacotes escaneados pela app podem vazar para cenarios in-process mesmo quando o comportamento real de producao usa child process com runtime isolado.
- A combinacao de teste in-process (detectar vazamento) e IT empacotada (validar caminho real) foi essencial para separar sintoma local de garantia de comportamento real.
- A propriedade `seed4j.cli.runtime.mode` e um bom guard clause para fixture de extensao porque conversa diretamente com o contrato ja usado no bootstrap.
