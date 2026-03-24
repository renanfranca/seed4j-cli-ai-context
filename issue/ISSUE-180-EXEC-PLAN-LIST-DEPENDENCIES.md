# Implementar coluna `Dependencies` no `seed4j list` com slices verticais

Este ExecPlan é um documento vivo. Atualize `Progress`, `Decisions`, `Risks` e `Lessons Learned` durante a execução.

## Purpose / Big Picture

Usuários hoje enxergam apenas slug e descrição no `seed4j list`, sem contexto de pré-requisitos. O objetivo é tornar o comando autoexplicativo, exibindo dependências tipadas (`module:` e `feature:`) com boa legibilidade no terminal, sem alterar o fluxo funcional de `seed4j apply`. O resultado deve ser observável diretamente pela saída do CLI em `standard mode` e `extension mode`.

## Scope

Escopo:

- Adicionar coluna `Dependencies` ao `seed4j list`.
- Construir dependências a partir de `modules.resources()`.
- Exibir dependências tipadas com marcador `(hidden)` quando o alvo de dependência de módulo estiver oculto.
- Definir largura/wrap determinísticos para terminal (sem truncar conteúdo).
- Atualizar a descrição do comando `list` para explicitar que a saída inclui dependências dos módulos.
- Atualizar testes e documentação de comandos.

Fora de escopo:

- Autoaplicação de dependências.
- Bloqueio de ordem de aplicação em `apply`.
- Alterações no comportamento funcional de `seed4j apply`.
- Mudança no contrato de configuração externa além da documentação da nova saída.

## Definitions

- `Seed4JModulesApplicationService`: serviço de aplicação usado pela CLI para obter catálogo de módulos.
- `resources()`: coleção de módulos já filtrada pelo comportamento de hidden resources.
- `Dependency token`: string de uma dependência no formato `module:<slug>` ou `feature:<slug>`.
- `Hidden marker`: sufixo textual ` (hidden)` adicionado ao token quando o alvo de uma dependência do tipo `MODULE` está oculto.
- `Vertical Slice`: incremento completo e verificável de ponta a ponta (código + teste + comportamento observável).

## Existing Context

- Implementação atual do list: `src/main/java/com/seed4j/cli/command/infrastructure/primary/ListModulesCommand.java`.
- O `list` hoje imprime apenas slug e descrição, com alinhamento por tamanho de slug.
- O `apply` usa `modules.resources()` para gerar subcomandos e não deve mudar: `src/main/java/com/seed4j/cli/command/infrastructure/primary/ApplyModuleCommand.java`.
- Testes de comando atuais: `src/test/java/com/seed4j/cli/command/infrastructure/primary/Seed4JCommandsFactoryTest.java`.
- Teste de comportamento de extension mode para `list`: `src/test/java/com/seed4j/cli/bootstrap/domain/ExtensionRuntimeBootstrapListPackagedJarIT.java`.
- Documentação atual do comando: `documentation/Commands.md`.

## Desired End State

- `seed4j list` passa a exibir colunas `Module`, `Dependencies`, `Description`.
- Dependências são derivadas de `modules.resources()` e preservam tipo (`module:`/`feature:`).
- Quando o alvo de dependência `MODULE` estiver oculto, token recebe ` (hidden)`.
- Coluna `Dependencies` usa largura efetiva `min(maxNaturalWidth, 60)` e quebra de linha sem truncamento.
- Ordenação alfabética de módulos permanece.
- `extension mode` continua aditivo e sem duplicidade de slugs.
- Ajuda do CLI reflete descrição do comando `list` mencionando módulos e dependências.
- Documentação oficial reflete o novo formato.

## Milestones

### Milestone 1 - VS1: Estruturar saída em 3 colunas com fallback `-`

#### Goal

Entregar a nova estrutura visual (`Module`, `Dependencies`, `Description`) mantendo o comportamento atual para módulos sem dependências.

#### Changes

- [ ] Editar `src/main/java/com/seed4j/cli/command/infrastructure/primary/ListModulesCommand.java` para:
- [ ] Imprimir cabeçalho de 3 colunas.
- [ ] Exibir `Dependencies` como `-` por padrão (ainda sem materializar tokens reais).
- [ ] Preservar ordenação por slug.
- [ ] Atualizar `spec().usageMessage().description(...)` do comando `list` para mencionar dependências.
- [ ] Introduzir tipos dedicados para formatação (TDD de tipos), por exemplo:
- [ ] `ListModuleRow` (slug, dependenciesText, description).
- [ ] `ListColumnsLayout` (larguras e padding).
- [ ] Criar teste focado no novo layout base em `src/test/java/com/seed4j/cli/command/infrastructure/primary/Seed4JCommandsFactoryTest.java`.

#### Validation

- [ ] Command: `./mvnw -Dtest=Seed4JCommandsFactoryTest test`
- [ ] Expected result: teste de `list` valida presença do cabeçalho, coluna `Dependencies` com `-` e descrição do comando atualizada no help.
- [ ] Command: `./mvnw test`
- [ ] Expected result: suíte unit/integration local passa sem regressão.

#### Acceptance Criteria

- [ ] Saída do `list` contém 3 colunas visíveis.
- [ ] Módulos sem dependência exibem `-`.
- [ ] Ordem alfabética de módulos é mantida.
- [ ] Descrição do comando `list` deixa explícito que também lista dependências.

### Milestone 2 - VS2: Dependências tipadas reais a partir de `resources()`

#### Goal

Substituir fallback por dependências reais tipadas (`module:`/`feature:`), sem mudar fonte de dados.

#### Changes

- [ ] Em `ListModulesCommand`, mapear dependências de `moduleResource.organization().dependencies()`.
- [ ] Gerar token por item:
- [ ] `module:<slug>` quando tipo for `MODULE`.
- [ ] `feature:<slug>` quando tipo for `FEATURE`.
- [ ] Manter ordem original das dependências retornadas pelo catálogo.
- [ ] Concatenar múltiplas dependências com `, `.
- [ ] Cobrir cenários com múltiplas dependências em teste de comando.

#### Validation

- [ ] Command: `./mvnw -Dtest=Seed4JCommandsFactoryTest test`
- [ ] Expected result: teste valida tokens tipados e separação por vírgula.
- [ ] Command: `./mvnw test`
- [ ] Expected result: sem regressão em `apply` e demais comandos.

#### Acceptance Criteria

- [ ] Dependências aparecem tipadas (`module:`/`feature:`).
- [ ] Múltiplas dependências usam `, `.
- [ ] Fonte de dados permanece `modules.resources()`.

### Milestone 3 - VS3: Hidden marker + largura máxima e wrap determinístico

#### Goal

Entregar legibilidade robusta em terminal para linhas longas e visibilidade de dependência oculta.

#### Changes

- [ ] Implementar regra de hidden marker no token apenas para dependências do tipo `MODULE`: `... (hidden)` quando slug da dependência não estiver entre slugs visíveis do próprio `resources()`.
- [ ] Implementar layout da coluna `Dependencies`:
- [ ] largura natural = maior entre cabeçalho e maior célula de dependências.
- [ ] largura efetiva = `min(larguraNatural, 60)`.
- [ ] wrap sem truncamento, preferindo quebra em `, `.
- [ ] linhas de continuação sem repetir `Module` e `Description`.
- [ ] Adicionar testes para:
- [ ] linha com dependências longas (wrap).
- [ ] presença de ` (hidden)` para dependência de `MODULE` oculto, sem marcar dependências de `FEATURE`.

#### Validation

- [ ] Command: `./mvnw -Dtest=Seed4JCommandsFactoryTest test`
- [ ] Expected result: layout com wrap e hidden marker validado por assert explícito.
- [ ] Command: `./mvnw test`
- [ ] Expected result: nenhuma regressão em comandos existentes.

#### Acceptance Criteria

- [ ] Não há truncamento de dependências.
- [ ] Wrap é determinístico.
- [ ] Dependência oculta de `MODULE` recebe sufixo ` (hidden)`.

### Milestone 4 - VS4: Compatibilidade extension mode + documentação + validação final

#### Goal

Fechar comportamento fim-a-fim em extension mode e atualizar documentação oficial.

#### Changes

- [ ] Ajustar testes em `src/test/java/com/seed4j/cli/bootstrap/domain/ExtensionRuntimeBootstrapListPackagedJarIT.java` para o novo formato de saída do `list`.
- [ ] Garantir que regras aditivas de extension mode permanecem (sem duplicidade, catálogo padrão preservado).
- [ ] Atualizar `documentation/Commands.md` seção de `seed4j list` com coluna `Dependencies`, tipagem, `-`, ` (hidden)` e regra de legibilidade.
- [ ] Revisar se há impacto em `README.md`; atualizar apenas se necessário para evitar drift de documentação.

#### Validation

- [ ] Command: `./mvnw -Dit.test=ExtensionRuntimeBootstrapListPackagedJarIT failsafe:integration-test failsafe:verify`
- [ ] Expected result: testes de list em JAR empacotado passam com novo formato.
- [ ] Command: `./mvnw clean verify`
- [ ] Expected result: validação completa verde (unit + integration + checkstyle + coverage).
- [ ] Command: `npm run prettier:check`
- [ ] Expected result: documentação e código formatados.

#### Acceptance Criteria

- [ ] `extension mode` continua aditivo e sem slugs duplicados.
- [ ] Documentação oficial descreve fielmente a nova saída.
- [ ] Pipeline de validação local completo passa.

## Vertical Slice Strategy

A estratégia de slices minimiza risco funcional e de layout:

1. VS1 muda estrutura visual sem risco semântico.
2. VS2 adiciona semântica de domínio (tipos de dependência).
3. VS3 resolve casos de borda de terminal e hidden marker.
4. VS4 fecha compatibilidade de runtime empacotado e documentação.

Cada slice gera comportamento observável no terminal e cobertura de teste antes do próximo.

## Progress

- [ ] Milestone 1 started
- [ ] Milestone 1 completed
- [ ] Milestone 2 started
- [ ] Milestone 2 completed
- [ ] Milestone 3 started
- [ ] Milestone 3 completed
- [ ] Milestone 4 started
- [ ] Milestone 4 completed

## Decisions

- Decision: Dependências serão sempre derivadas de `modules.resources()`.
  Rationale: mantém consistência com catálogo efetivo da CLI e com filtros de hidden já aplicados.
  Date/Author: 2026-03-23 / Codex

- Decision: Dependência oculta de `MODULE` será exibida com sufixo ` (hidden)`.
  Rationale: preserva informação de domínio sem esconder relacionamento real e evita marcar `FEATURE` sem contrato de ocultação.
  Date/Author: 2026-03-23 / Codex

- Decision: Coluna `Dependencies` terá limite de largura 60 com wrap sem truncamento.
  Rationale: mantém legibilidade em terminal sem perder informação.
  Date/Author: 2026-03-23 / Codex

## Risks and Mitigations

- Risk: Regressão no parser de testes de saída em JAR empacotado.
  Mitigation: ajustar regex/asserts no IT específico do `list` e validar com failsafe direcionado.

- Risk: Wrap quebrar alinhamento e tornar saída ambígua.
  Mitigation: encapsular cálculo de layout em tipo dedicado e cobrir com testes de linhas longas.

- Risk: Hidden marker ser interpretado incorretamente como bug obrigatório.
  Mitigation: documentar explicitamente que marcador é informativo e depende do conjunto visível do catálogo.

- Risk: Misturar FEATURE/MODULE e perder semântica.
  Mitigation: mapear tokens por tipo de dependência com teste dedicado para cada tipo.

## Validation Strategy

1. Rodar testes focados em `list` a cada slice.
2. Rodar `./mvnw test` ao final de cada milestone para evitar regressões acumuladas.
3. Rodar IT de extension mode antes do fechamento.
4. Rodar `./mvnw clean verify` como gate final.
5. Rodar `npm run prettier:check` para validar documentação e estilo.

## Rollout and Recovery

Rollout:

1. Entregar em commits por slice (um commit por milestone) para facilitar revisão e rollback.
2. Abrir PR com evidência de saída `seed4j list` antes/depois.

Recovery:

1. Se layout causar regressão grave, reverter apenas VS3 (wrap) mantendo VS2 (semântica).
2. Se IT de extension mode falhar por parsing, isolar ajuste no teste sem reverter comportamento funcional.
3. Em último caso, voltar para formato anterior e manter plano aberto com status parcial.

## Lessons Learned

- [ ] Preencher durante execução com descobertas reais (ex.: particularidades de dependências hidden por tag/slug no core).
