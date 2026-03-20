# Sonar Cleanup: 16 Open Issues (2026-03-20)

This ExecPlan is a living document. Keep `Progress`, `Decisions`, `Risks`, and `Lessons Learned` up to date as work advances.

## Purpose / Big Picture

Fechar as 16 issues abertas no Sonar mostradas no snapshot de 2026-03-20, sem mudar comportamento funcional do CLI. O resultado observavel esperado e: build local verde (`clean verify`) e nenhum alerta remanescente desse lote apos nova analise do SonarQube. O foco e manutencao, legibilidade e padronizacao de testes.

## Scope

Em escopo:

- Corrigir as 16 issues listadas no snapshot (producao e testes).
- Preservar comportamento atual do runtime bootstrap e comandos.
- Reorganizar testes repetitivos em parameterized tests apenas onde os cenarios sao homogeneos.
- Executar validacoes focadas e validacao completa.

Fora de escopo:

- Mudancas de regra de negocio.
- Refactors amplos fora dos pontos sinalizados pelo Sonar.
- Fechar issues antigas que nao estao nesse snapshot.

## Definitions

- Unnamed pattern em catch: uso de `_` em `catch` quando a excecao nao e utilizada.
- Assertion chain: consolidar multiplos `assertThat(...)` sobre o mesmo alvo em uma cadeia unica.
- Parameterized test: um `@ParameterizedTest` com `@MethodSource` para varios cenarios de mesma forma.
- Snapshot Sonar: lista de issues fornecida no HTML, com 16/16 visiveis.

## Existing Context

Issues do snapshot agrupadas por arquivo:

- `src/main/java/com/seed4j/cli/Seed4JCliApp.java`
  - L81, L148, L150: `Replace "e" with an unnamed pattern.`
- `src/main/java/com/seed4j/cli/bootstrap/domain/CliVersion.java`
  - L42: `Replace "e" with an unnamed pattern.`
- `src/main/java/com/seed4j/cli/bootstrap/domain/RuntimeModeConfigReader.java`
  - L23: `Define a constant instead of duplicating this literal "seed4j" 3 times.`
  - L43: `Extract this nested try block into a separate method.`
  - L45, L50: `Replace "e" with an unnamed pattern.`
- `src/main/java/com/seed4j/cli/bootstrap/domain/Seed4JCliLauncher.java`
  - L66: `Replace "e" with an unnamed pattern.`
- `src/test/java/com/seed4j/cli/bootstrap/domain/ExtensionRuntimeBootstrapListPackagedJarIT.java`
  - L37, L52: `Join these multiple assertions subject to one assertion chain.`
- `src/test/java/com/seed4j/cli/bootstrap/domain/ExtensionRuntimeFixture.java`
  - L70: `Remove this unused "ignored" local variable.`
- `src/test/java/com/seed4j/cli/bootstrap/domain/ExtensionRuntimeFixtureTest.java`
  - L57: `Join these multiple assertions subject to one assertion chain.`
- `src/test/java/com/seed4j/cli/bootstrap/domain/RuntimeSelectionTest.java`
  - L290: `Replace these 7 tests with a single Parameterized one.`
- `src/test/java/com/seed4j/cli/bootstrap/domain/Seed4JCliLauncherTest.java`
  - L165: `Replace these 5 tests with a single Parameterized one.`
  - L232: `Replace these 4 tests with a single Parameterized one.`

Testes ja existentes que cobrem o impacto:

- `src/test/java/com/seed4j/cli/Seed4JCliAppTest.java`
- `src/test/java/com/seed4j/cli/bootstrap/domain/RuntimeSelectionTest.java`
- `src/test/java/com/seed4j/cli/bootstrap/domain/Seed4JCliLauncherTest.java`
- `src/test/java/com/seed4j/cli/bootstrap/domain/ExtensionRuntimeFixtureTest.java`
- `src/test/java/com/seed4j/cli/bootstrap/domain/ExtensionRuntimeBootstrapListPackagedJarIT.java`

## Desired End State

- As 16 issues do snapshot estao corrigidas.
- Nao ha regressao funcional em bootstrap standard/extension.
- Testes continuam legiveis, com Given/When/Then claro e sem esconder asserts criticos em helper.
- `./mvnw clean verify` passa.
- A analise SonarQube subsequente nao reabre os mesmos apontamentos.

## Milestones

### Milestone 1 - Baseline e guardrails

#### Goal

Estabelecer baseline executavel e garantir que os proximos refactors sao estritamente de manutencao.

#### Changes

- [x] Registrar no PR/branch o inventario final de 16 issues (arquivo + linha + regra textual).
- [x] Confirmar que todas as alteracoes previstas sao no-op funcional.
- [x] Definir estrategia de commits por bloco: producao, testes simples, parameterizacao.

#### Validation

- [x] Command: `./mvnw -Dtest='Seed4JCliAppTest,RuntimeSelectionTest,Seed4JCliLauncherTest,ExtensionRuntimeFixtureTest' test`
- [x] Expected result: baseline de testes unitarios verde antes das alteracoes.

#### Acceptance Criteria

- [x] Inventario das 16 issues confirmado no plano.
- [x] Baseline de testes focados verde.

### Milestone 2 - Correcao de producao (8 issues)

#### Goal

Eliminar issues de producao sem alterar semantica de runtime.

#### Changes

- [x] Editar `src/main/java/com/seed4j/cli/Seed4JCliApp.java`:
  - substituir catches com variavel nao usada por unnamed pattern.
- [x] Editar `src/main/java/com/seed4j/cli/bootstrap/domain/CliVersion.java`:
  - substituir catch com variavel nao usada por unnamed pattern.
- [x] Editar `src/main/java/com/seed4j/cli/bootstrap/domain/Seed4JCliLauncher.java`:
  - substituir catch com variavel nao usada por unnamed pattern.
- [x] Editar `src/main/java/com/seed4j/cli/bootstrap/domain/RuntimeModeConfigReader.java`:
  - extrair literal `seed4j` para constante.
  - extrair bloco `try` aninhado em metodo dedicado para leitura/normalizacao do modo.
  - substituir catches com variavel nao usada por unnamed pattern.
  - manter mensagens de erro atuais.

#### Validation

- [x] Command: `./mvnw -Dtest='Seed4JCliAppTest,Seed4JCliLauncherTest,RuntimeSelectionTest' test`
- [x] Expected result: comportamento de bootstrap permanece inalterado, testes verdes.

#### Acceptance Criteria

- [x] Nenhuma issue de producao desse lote permanece.
- [x] Sem mudanca de mensagem de erro observavel fora do esperado.

### Milestone 3 - Correcao de testes diretos (4 issues)

#### Goal

Fechar issues em testes sem alterar intencao dos cenarios.

#### Changes

- [x] Editar `src/test/java/com/seed4j/cli/bootstrap/domain/ExtensionRuntimeBootstrapListPackagedJarIT.java`:
  - consolidar asserts por alvo em assertion chain unica nos pontos sinalizados.
- [x] Editar `src/test/java/com/seed4j/cli/bootstrap/domain/ExtensionRuntimeFixtureTest.java`:
  - consolidar assert chain no teste sinalizado.
- [x] Editar `src/test/java/com/seed4j/cli/bootstrap/domain/ExtensionRuntimeFixture.java`:
  - remover variavel local nao utilizada no `try-with-resources` do jar minimo.

#### Validation

- [x] Command: `./mvnw -Dtest='ExtensionRuntimeFixtureTest' test`
- [x] Expected result: fixture e asserts continuam validos.
- [x] Command: `./mvnw -Dtest='NoSuchTest' -Dsurefire.failIfNoSpecifiedTests=false -Dit.test='*ExtensionRuntimeBootstrapListPackagedJarIT*' -DfailIfNoTests=false verify`
- [x] Expected result: IT empacotado segue verde.

#### Acceptance Criteria

- [x] Todas as 4 issues de testes diretos fechadas.
- [x] Nenhuma regressao no fluxo empacotado `list`.

### Milestone 4 - Parameterizacao dos testes repetitivos (4 issues)

#### Goal

Reduzir duplicacao mantendo legibilidade e diagnostico claro de falha por cenario.

#### Changes

- [x] Editar `src/test/java/com/seed4j/cli/bootstrap/domain/RuntimeSelectionTest.java`:
  - consolidar o grupo de 7 testes semelhantes em 1 `@ParameterizedTest` com casos nomeados.
  - manter Given/When/Then e assert explicito no corpo do teste.
- [x] Editar `src/test/java/com/seed4j/cli/bootstrap/domain/Seed4JCliLauncherTest.java`:
  - consolidar o grupo de 5 testes semelhantes em 1 `@ParameterizedTest`.
  - consolidar o grupo de 4 testes semelhantes em 1 `@ParameterizedTest`.
  - manter cenarios heterogeneos fora da parameterizacao (sem misturar contratos diferentes).

#### Validation

- [x] Command: `./mvnw -Dtest='RuntimeSelectionTest,Seed4JCliLauncherTest' test`
- [x] Expected result: suites passam com cobertura dos mesmos cenarios originais.

#### Acceptance Criteria

- [x] 3 apontamentos de "Replace these N tests..." fechados.
- [x] Nomes dos casos parameterizados permitem identificar o cenario que falhou sem ambiguidade.

### Milestone 5 - Fechamento e revalidacao completa

#### Goal

Consolidar a entrega e garantir que o lote esta pronto para merge.

#### Changes

- [x] Executar formatacao apenas se necessario (`npm run prettier:format`).
- [x] Revisar diff para confirmar ausencia de mudanca funcional inesperada.
- [x] Executar validacao completa do repositorio.
- [x] Rodar nova analise SonarQube no fluxo oficial (local ou CI) e confirmar fechamento das 16 issues.

#### Validation

- [x] Command: `./mvnw clean verify`
- [x] Expected result: surefire/failsafe/checkstyle/jacoco verdes.
- [x] Command: `<pipeline ou comando oficial de sonar da equipe>`
- [x] Expected result: as 16 issues deste snapshot aparecem como resolvidas.

#### Acceptance Criteria

- [x] Build completo verde.
- [x] Sem reabertura das issues deste lote no Sonar.

## Progress

- [x] Milestone 1 started
- [x] Milestone 1 completed
- [x] Milestone 2 started
- [x] Milestone 2 completed
- [x] Milestone 3 started
- [x] Milestone 3 completed
- [x] Milestone 4 started
- [x] Milestone 4 completed
- [x] Milestone 5 started
- [x] Milestone 5 completed

## Decisions

- Decision: aplicar unnamed pattern em catches sem uso, em vez de renomear variaveis para `ignored`.
  Rationale: fecha regra java22 do Sonar com menor ruido.
  Date/Author: 2026-03-20 / Codex

- Decision: parameterizar apenas testes com setup/assert estruturalmente iguais.
  Rationale: evita perda de legibilidade e respeita o guideline local de asserts explicitos.
  Date/Author: 2026-03-20 / Codex

- Decision: manter mensagens de erro existentes em `RuntimeModeConfigReader` durante o refactor.
  Rationale: reduzir risco de regressao em testes e em diagnostico para usuario final.
  Date/Author: 2026-03-20 / Codex

- Decision: unificar toda a validacao de `extensionJarEntries` em uma unica assertion chain em `ExtensionRuntimeFixtureTest`.
  Rationale: a primeira tentativa manteve duas chains separadas e o Sonar continuou apontando `java:S5853`; a cadeia unica fecha o apontamento sem mudar semantica.
  Date/Author: 2026-03-20 / Codex

## Risks and Mitigations

- Risk: parameterizacao excessiva reduzir clareza de falhas.
  Mitigation: usar casos nomeados e separar por contrato de comportamento.

- Risk: refactor de `RuntimeModeConfigReader` alterar fluxo de erro.
  Mitigation: preservar mensagens e rodar `Seed4JCliLauncherTest` completo apos mudanca.

- Risk: suporte de ferramentas para unnamed pattern (`_`) variar por ambiente.
  Mitigation: validar compilacao/testes imediatamente apos alterar catches.

- Risk: Sonar apontar novas issues derivadas dos mesmos arquivos apos refactor.
  Mitigation: fechar em slices pequenos e reexecutar validacao entre milestones.

## Validation Strategy

1. Rodar baseline focado antes de editar.
2. Aplicar e validar correcoes de producao.
3. Aplicar e validar correcoes de testes diretos.
4. Aplicar parameterizacao e validar suites afetadas.
5. Rodar `./mvnw clean verify`.
6. Publicar nova analise SonarQube e comparar com o snapshot de 16 issues.

## Rollout and Recovery

- Rollout: merge normal em branch principal apos validacao completa e sonar atualizado.
- Recovery: se algum refactor de teste gerar ruido, reverter apenas o bloco de parameterizacao e manter correcoes seguras de producao no mesmo lote ou em lote separado.

## Lessons Learned

- A execucao do scanner Sonar em sandbox falhou com `Operation not permitted`; para esse projeto e necessario rodar o fluxo com Docker/Sonar fora do sandbox.
- A analise Sonar e assincrona (`/api/ce/task`); validar conclusao exige esperar status `SUCCESS` antes de consultar issues via API.
- O apontamento de assertion chain em `ExtensionRuntimeFixtureTest` exigia encadear tambem o `contains(...)` inicial na mesma cadeia para remover `java:S5853`.
