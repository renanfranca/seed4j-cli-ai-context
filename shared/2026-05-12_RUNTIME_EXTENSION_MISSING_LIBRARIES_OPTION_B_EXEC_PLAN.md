# RuntimeExtensionMissingLibrariesSelector - Option B (decisao centralizada por biblioteca)

Este ExecPlan e um documento vivo. Mantenha `Progress`, `Decisions`, `Risks` e `Lessons Learned` atualizados conforme a execucao.

Safety boundary: Esta tarefa e limitada a manutencao defensiva e autorizada deste repositorio para reduzir risco de regressao de runtime no `extension mode`.

## Purpose / Big Picture

Hoje a regra de selecao de `libs ausentes` depende da ordem de execucao dentro de `select(...)`: um fail-fast acontece antes da logica de `missing`, criando acoplamento temporal. A mudanca da Opcao B centraliza a decisao por biblioteca em um unico ponto, retornando um estado explicito (`MISSING`, `PRESENT` ou `CONFLICT`) para cada entrada da extensao. O comportamento observavel esperado e: conflitos continuam falhando com mensagem clara, bibliotecas realmente ausentes continuam entrando no `loader.path`, e manutencoes futuras deixam de depender de pre-condicao implicita de ordem.

## Scope

Em escopo:

- Refatorar `RuntimeExtensionMissingLibrariesSelector` para decisao unica por biblioteca da extensao.
- Remover dependencia comportamental entre `failWhenMissingIdentityShadowsCliLibrary(...)` e `missingFrom(...)`.
- Ajustar/expandir testes unitarios para provar o novo contrato sem acoplamento temporal.
- Revalidar impacto no fluxo que monta `loader.path` em `RuntimeExtensionLoaderPathResolver`.

Fora de escopo:

- Alteracoes de arquitetura fora de `bootstrap/domain` (por exemplo worker separado, mudanca de launcher).
- Alterar contrato funcional de alto nivel de `extension mode` alem da regra de decisao de biblioteca.
- Mudancas de comportamento em regras nao relacionadas (ex.: cache de overlay, descoberta de `Start-Class`).

## Definitions

- `acoplamento temporal`: quando a regra correta depende da ordem entre passos separados, em vez de ser garantida no mesmo ponto de decisao.
- `identidade de biblioteca`: `RuntimeLibraryIdentity`, representada por `coordinate + version`.
- `biblioteca da extensao`: item de `BOOT-INF/lib` do JAR de extensao, modelado como `RuntimeLibraryEntry`.
- `biblioteca ausente`: biblioteca da extensao que precisa ser adicionada ao `loader.path` porque nao esta presente no runtime do CLI.
- `conflito`: cenario que deve interromper o fluxo com `InvalidRuntimeConfigurationException`.
- `decisao por biblioteca`: resultado explicito da avaliacao de uma biblioteca da extensao contra o contexto de bibliotecas do CLI (`MISSING`, `PRESENT`, `CONFLICT`).

## Existing Context

- Arquivo principal com acoplamento temporal atual:
  - `src/main/java/com/seed4j/cli/bootstrap/domain/RuntimeExtensionMissingLibrariesSelector.java`
- Hoje `select(...)` executa:
  1. `failWhenMissingIdentityShadowsCliLibrary(...)`
  2. `ensureNoVersionConflict(...)`
  3. `missingFrom(...)` (com fallback por `fileName` em `orElseGet(...)`)
- Essa composicao cria dependencia de ordem: se alguem reordenar/remover passo anterior, o comportamento de seguranca muda sem sinal local.
- Testes diretamente relacionados:
  - `src/test/java/com/seed4j/cli/bootstrap/domain/RuntimeExtensionMissingLibrariesSelectorTest.java`
  - `src/test/java/com/seed4j/cli/bootstrap/domain/RuntimeExtensionLoaderPathResolverTest.java`
- O fluxo consumidor da selecao:
  - `src/main/java/com/seed4j/cli/bootstrap/domain/RuntimeExtensionLoaderPathResolver.java`

## Desired End State

- `RuntimeExtensionMissingLibrariesSelector` decide cada biblioteca da extensao em um unico ponto logico.
- O algoritmo nao depende de pre-validacao externa para garantir regras de conflito.
- `select(...)` apenas:
  - agrega nomes de arquivo para decisoes `MISSING`;
  - interrompe ao encontrar `CONFLICT` com mensagem deterministica.
- Testes documentam explicitamente os tres estados de decisao e os cenarios de fallback por `fileName` somente quando identidade estiver ausente.
- O comportamento externo de `loader.path` permanece estavel para cenarios ja validados.

## Milestones

### Milestone 1 - Modelar decisao unica por biblioteca

#### Goal

Introduzir um modelo explicito de decisao para eliminar a dependencia de ordem entre validacao e filtro de ausentes.

#### Changes

- [ ] Editar `src/main/java/com/seed4j/cli/bootstrap/domain/RuntimeExtensionMissingLibrariesSelector.java` para introduzir o conceito de decisao por biblioteca (por exemplo enum/record interno com `MISSING`, `PRESENT`, `CONFLICT`).
- [ ] Substituir o encadeamento atual (`failWhenMissingIdentityShadowsCliLibrary` + `missingFrom`) por uma unica funcao decisora que recebe:
  - biblioteca da extensao;
  - contexto das bibliotecas do CLI (identidades e nomes de arquivo);
  - mapa de versoes por coordenada do CLI.
- [ ] Manter fail-fast para `CONFLICT`, mas decidido dentro da mesma funcao de avaliacao por biblioteca.
- [ ] Preservar mensagens de erro ja consolidadas, ajustando apenas quando necessario para refletir o novo contrato.

#### Validation

- [ ] Command: `./mvnw -Dtest=RuntimeExtensionMissingLibrariesSelectorTest test`
- [ ] Expected result: testes atuais continuam verdes ou falham apenas nos pontos onde o contrato mudou e precisa de ajuste explicito.

#### Acceptance Criteria

- [ ] Nao existe mais dependencia funcional entre uma pre-validacao separada e a regra de ausente.
- [ ] Cada biblioteca da extensao tem resultado deterministico derivado de uma unica funcao decisora.
- [ ] Conflitos continuam falhando com `InvalidRuntimeConfigurationException`.

### Milestone 2 - Endurecer testes do contrato de decisao

#### Goal

Cobrir de forma direta os cenarios `MISSING`, `PRESENT` e `CONFLICT`, incluindo os casos de fallback por `fileName` sem identidade.

#### Changes

- [ ] Atualizar `src/test/java/com/seed4j/cli/bootstrap/domain/RuntimeExtensionMissingLibrariesSelectorTest.java` com cenarios nomeados por regra de negocio (nao por ordem de execucao interna).
- [ ] Garantir cobertura de:
  - identidade igual ao CLI -> `PRESENT`;
  - identidade diferente (sem conflito de coordenada/versao) -> `MISSING`;
  - mesma coordenada com versao divergente -> `CONFLICT`;
  - sem identidade e `fileName` colidindo com CLI -> `CONFLICT`;
  - sem identidade e `fileName` nao colidindo -> `MISSING`.
- [ ] Evitar testes que assumam detalhes de implementacao (por exemplo ordem de chamadas internas), mantendo foco no contrato observavel.

#### Validation

- [ ] Command: `./mvnw -Dtest=RuntimeExtensionMissingLibrariesSelectorTest test`
- [ ] Expected result: suite do seletor verde com regras explicitas da Opcao B.

#### Acceptance Criteria

- [ ] Os testes descrevem o contrato funcional sem depender de acoplamento temporal.
- [ ] O branch de fallback por `fileName` fica coberto por cenarios legitimos do contrato novo.

### Milestone 3 - Validacao integrada com resolver e suite local

#### Goal

Garantir que a refatoracao nao alterou indevidamente o comportamento de `loader.path` no fluxo consumidor.

#### Changes

- [ ] Executar e ajustar, se necessario, `src/test/java/com/seed4j/cli/bootstrap/domain/RuntimeExtensionLoaderPathResolverTest.java` para refletir o contrato final do seletor.
- [ ] Revisar mensagens de falha esperadas nos testes integradores de dominio quando houver alteracao textual inevitavel.
- [ ] Confirmar que nenhum trecho legado morto do seletor permaneceu apos centralizacao da decisao.

#### Validation

- [ ] Command: `./mvnw -Dtest=RuntimeExtensionMissingLibrariesSelectorTest,RuntimeExtensionLoaderPathResolverTest test`
- [ ] Expected result: regras do seletor e montagem de `loader.path` verdes em conjunto.
- [ ] Command: `./mvnw clean verify`
- [ ] Expected result: validacao completa local verde (tests, coverage, checkstyle e gates do projeto).

#### Acceptance Criteria

- [ ] `loader.path` continua incluindo apenas libs realmente ausentes da extensao.
- [ ] Cenarios de conflito continuam abortando com diagnostico claro.
- [ ] Sem regressao no comportamento observado do `extension mode` coberto pelos testes atuais.

## Progress

- [ ] Milestone 1 started
- [ ] Milestone 1 completed
- [ ] Milestone 2 started
- [ ] Milestone 2 completed
- [ ] Milestone 3 started
- [ ] Milestone 3 completed

## Decisions

- Decision: Adotar a Opcao B (decisao centralizada por biblioteca) em vez de apenas documentar ordem (Opcao A).
  Rationale: reduz risco estrutural de regressao por manutencao futura e remove pre-condicao implicita entre metodos.
  Date/Author: 2026-05-12 / User + Codex

- Decision: Manter semantica externa de selecao (`lista final de fileNames` e fail-fast em conflito) durante a refatoracao.
  Rationale: melhorar robustez interna sem alterar contrato publico esperado pelo `RuntimeExtensionLoaderPathResolver`.
  Date/Author: 2026-05-12 / Codex

## Risks and Mitigations

- Risk: alterar mensagem de erro e quebrar testes por detalhe textual sem ganho funcional.
  Mitigation: preservar textos atuais quando possivel; quando mudar, atualizar testes com justificativa explicita no commit.

- Risk: regressao silenciosa em caso sem identidade (fallback por `fileName`).
  Mitigation: cobrir explicitamente colisao e nao-colisao de `fileName` em testes dedicados.

- Risk: refatoracao remover validacao de conflito por coordenada/versao acidentalmente.
  Mitigation: manter testes de conflito existentes e adicionar casos de fronteira no milestone 2.

- Risk: ganho de design sem ganho observavel para usuario final.
  Mitigation: validar fluxo consumidor (`RuntimeExtensionLoaderPathResolverTest`) e confirmar resultado de `loader.path`.

## Validation Strategy

1. Rodar testes focados no seletor: `./mvnw -Dtest=RuntimeExtensionMissingLibrariesSelectorTest test`.
2. Rodar testes focados no consumidor de `loader.path`: `./mvnw -Dtest=RuntimeExtensionMissingLibrariesSelectorTest,RuntimeExtensionLoaderPathResolverTest test`.
3. Rodar validacao completa do repositorio: `./mvnw clean verify`.

## Rollout and Recovery

Rollout:

- Entregar a mudanca em commit unico focado no seletor + testes.
- Validar em CI com o mesmo comando baseline (`./mvnw clean verify`).

Recovery:

- Se houver regressao, reverter apenas o commit da refatoracao da Opcao B.
- Como fallback temporario, restaurar implementacao anterior orientada por ordem enquanto se corrige a decisao centralizada em branch separado.

## Lessons Learned

- A cobertura parcial no branch de fallback (`orElseGet`) sinalizou um risco real de design (acoplamento temporal), mesmo sem bug funcional imediato.
- Regras de conflito e regra de `missing` no mesmo ponto decisor tornam o contrato mais resiliente a reordenacoes e reuso futuro.
