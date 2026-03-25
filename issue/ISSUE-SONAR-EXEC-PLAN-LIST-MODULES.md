# Corrigir issues Sonar em `ListModulesCommand` e `ListModulesCommandTest`

Este ExecPlan é um documento vivo. Atualize `Progress`, `Decisions`, `Risks` e `Lessons Learned` durante a execução.

## Purpose / Big Picture

Existem 2 issues de maintainability abertos no Sonar para o comando `list`: um no algoritmo de wrap de dependências e outro em estilo de asserções no teste unitário. O objetivo é remover ambos sem alterar o comportamento funcional do CLI. O resultado deve ser observável por testes locais passando e por fechamento dos issues no Sonar após análise.

## Scope

Escopo:

- Refatorar o loop em `wrapDependencies(...)` para atender a regra Sonar de no máximo um `break/continue` por laço.
- Ajustar o teste em `ListModulesCommandTest` para usar uma única assertion chain no mesmo subject.
- Garantir não regressão de formatação da saída textual do comando `list`.
- Validar localmente com testes focados e validação completa.

Fora de escopo:

- Mudanças de feature no comando `list`.
- Alterações de contrato de domínio de módulos/dependências.
- Mudanças em documentação funcional do comando.

## Definitions

- `wrapDependencies`: método em `ListModulesCommand` responsável por quebrar a coluna `Dependencies` em múltiplas linhas.
- `Assertion chain`: encadeamento AssertJ no mesmo `assertThat(...)` para múltiplas verificações do mesmo objeto.
- `Hard wrap`: quebra forçada de token quando um único token excede a largura da coluna.

## Existing Context

- Issue 1 (Sonar): `src/main/java/com/seed4j/cli/command/infrastructure/primary/ListModulesCommand.java` em torno da linha 143, regra “Reduce the total number of break and continue statements in this loop to use at most one”.
- O loop atual em `wrapDependencies(...)` usa dois `continue`, o que dispara a regra.
- Issue 2 (Sonar): `src/test/java/com/seed4j/cli/command/infrastructure/primary/ListModulesCommandTest.java` na linha 46, regra “Join these multiple assertions subject to one assertion chain”.
- O teste `shouldWrapDependenciesColumnAtSixtyCharactersWithoutRepeatingDescription(...)` hoje repete `assertThat(output)` em chamadas separadas.

## Desired End State

- `wrapDependencies(...)` mantém o mesmo resultado de output e deixa de violar a regra de `break/continue`.
- O teste de wrap usa uma única chain AssertJ para as verificações no mesmo `output`.
- `ListModulesCommandTest` passa localmente.
- `./mvnw clean verify` passa.
- Sonar não retorna os 2 issues listados neste plano.

## Milestones

### Milestone 1 - Refatorar loop de wrap sem regressão

#### Goal

Remover o smell de controle de fluxo no laço de `wrapDependencies(...)`, preservando o comportamento atual de quebra por vírgula e hard wrap de token longo.

#### Changes

- [ ] Editar `src/main/java/com/seed4j/cli/command/infrastructure/primary/ListModulesCommand.java`.
- [ ] Reestruturar o fluxo interno do `for` para evitar múltiplos `continue` (preferindo `if/else` explícito e helpers pequenos).
- [ ] Manter semântica de:
- [ ] preenchimento de `currentLine` quando `candidateLine` cabe na largura.
- [ ] flush de `currentLine` antes de tratar token que não cabe.
- [ ] hard wrap quando token isolado excede a largura.
- [ ] adição final da última linha acumulada.
- [ ] Revisar se há possibilidade de linha vazia indevida em cenários limites.

#### Validation

- [ ] Command: `./mvnw -Dtest=ListModulesCommandTest test`
- [ ] Expected result: todos os testes de `ListModulesCommandTest` passam.
- [ ] Command: `./mvnw -Dtest=Seed4JCommandsFactoryTest test`
- [ ] Expected result: testes de fábrica de comandos continuam verdes (sem regressão na saída observada de `list`).

#### Acceptance Criteria

- [ ] O laço em `wrapDependencies(...)` não viola mais a regra Sonar de `break/continue`.
- [ ] O comportamento de wrap e hard wrap permanece equivalente ao comportamento antes do refactor.

### Milestone 2 - Unificar assertion chain no teste

#### Goal

Resolver o issue Sonar de assertividade no teste, melhorando legibilidade sem mudar cobertura comportamental.

#### Changes

- [ ] Editar `src/test/java/com/seed4j/cli/command/infrastructure/primary/ListModulesCommandTest.java`.
- [ ] No teste `shouldWrapDependenciesColumnAtSixtyCharactersWithoutRepeatingDescription(...)`, trocar múltiplos `assertThat(output)` por um único chain:
- [ ] `.containsPattern(...)`
- [ ] `.containsPattern(...)`
- [ ] `.doesNotContainPattern(...)`
- [ ] Revisar rapidamente o restante da classe para outros casos idênticos no mesmo padrão.

#### Validation

- [ ] Command: `./mvnw -Dtest=ListModulesCommandTest test`
- [ ] Expected result: classe de teste passa integralmente.

#### Acceptance Criteria

- [ ] O método de teste afetado usa uma única assertion chain no mesmo subject.
- [ ] O issue Sonar da linha 46 deixa de ocorrer.

### Milestone 3 - Verificação de fechamento no pipeline e Sonar

#### Goal

Garantir que o ajuste local é estável e que os issues realmente fecharam no Sonar.

#### Changes

- [ ] Executar validação completa do repositório.
- [ ] Rodar análise Sonar local seguindo o fluxo validado do projeto (Docker Sonar + `sonar:sonar`) quando necessário.
- [ ] Confirmar fechamento dos dois issues por arquivo/linha.

#### Validation

- [ ] Command: `./mvnw clean verify`
- [ ] Expected result: build verde com testes, checkstyle e cobertura.
- [ ] Command: `docker compose -f src/main/docker/sonar.yml up -d`
- [ ] Expected result: SonarQube disponível (`UP`) em `http://localhost:9001`.
- [ ] Command: `./mvnw clean verify sonar:sonar -Dsonar.token=<token>`
- [ ] Expected result: análise enviada com sucesso.
- [ ] Command: consulta de CE task e issues (API Sonar)
- [ ] Expected result: 0 issues abertos para os dois achados deste plano.

#### Acceptance Criteria

- [ ] Pipeline local completo passa.
- [ ] Sonar não mostra os 2 issues reportados no contexto inicial.

## Progress

- [ ] Milestone 1 started
- [ ] Milestone 1 completed
- [ ] Milestone 2 started
- [ ] Milestone 2 completed
- [ ] Milestone 3 started
- [ ] Milestone 3 completed

## Decisions

- Decision: priorizar refactor mínimo no loop, sem redesenhar contrato do comando.
  Rationale: o alvo é correção de maintainability com risco funcional baixo.
  Date/Author: 2026-03-24 / Codex

- Decision: manter verificação por regex nos testes existentes.
  Rationale: já valida formato textual completo com alinhamento e continuidades de linha.
  Date/Author: 2026-03-24 / Codex

- Decision: incluir validação Sonar no plano, além de `clean verify`.
  Rationale: `clean verify` não garante sozinho fechamento de issue Sonar.
  Date/Author: 2026-03-24 / Codex

## Risks and Mitigations

- Risk: refactor do loop alterar comportamento em casos limítrofes de largura (60 chars).
  Mitigation: manter testes de wrap e hard wrap existentes como rede de segurança e validar saída textual.

- Risk: regex do teste ficar permissiva demais e não detectar mudança de layout.
  Mitigation: manter padrões ancorados por linha (`(?m)^...$`) e checar linha de continuação sem descrição.

- Risk: falso positivo de “issue ainda aberto” por processamento assíncrono do Sonar.
  Mitigation: aguardar conclusão da CE task antes de consultar issues.

## Validation Strategy

1. Rodar testes focados (`ListModulesCommandTest`) após cada ajuste.
2. Rodar testes adjacentes (`Seed4JCommandsFactoryTest`) para regressão de integração no comando.
3. Rodar `./mvnw clean verify` como gate local final.
4. Rodar Sonar local e validar fechamento explícito dos 2 issues.

## Rollout and Recovery

Rollout:

1. Commit único com correções Sonar de baixo risco usando convenção `refactor(command): ...` ou `test(command): ...` conforme escopo final.
2. PR com evidência de testes e referência aos issue IDs do Sonar.

Recovery:

1. Se houver regressão de output, reverter apenas o refactor de `wrapDependencies(...)` e manter o ajuste de assertion chain.
2. Reaplicar refactor do loop em passos menores com testes entre cada passo.

## Lessons Learned

- Preencher durante execução.
