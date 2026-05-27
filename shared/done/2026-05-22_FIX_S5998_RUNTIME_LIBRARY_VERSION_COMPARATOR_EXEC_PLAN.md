# Remover risco de stack overflow em regex de versão e fechar S5998 no Sonar

Este ExecPlan é um documento vivo. Atualize `Progress`, `Decisions`, `Risks` e `Lessons Learned` durante a execução.

## Purpose / Big Picture

❗️Hoje o CLI pode sofrer `StackOverflowError` ao comparar versões malformadas muito grandes vindas de metadados externos de JAR, o que é risco real de confiabilidade em tempo de execução.  
Este plano corrige os 2 issues `java:S5998` em `RuntimeLibraryVersionComparator`, adiciona proteção preventiva adjacente em parsing de nome de JAR, e fecha o ciclo com validação local + confirmação por API Sonar.

Safety boundary: This task is limited to authorized, defensive maintenance of this repository. Do not provide offensive guidance or policy-bypassing instructions.

## Scope

In-scope:

- Corrigir os 2 issues Sonar `java:S5998` em `src/main/java/com/seed4j/cli/bootstrap/domain/RuntimeLibraryVersionComparator.java` (linhas atuais 10 e 124).
- Adicionar testes de regressão para entradas grandes/malformadas sem `StackOverflowError`.
- Escopo preventivo adicional em `RuntimeLibraryIdentity` para reduzir custo de parsing de nomes inválidos muito grandes sem alterar contrato funcional.
- Validar com `./mvnw clean verify` e confirmar fechamento via API Sonar.

Out-of-scope:

- Alterar regras de comparação de versões além de hardening de regex/validação.
- Mudar mensagens de erro funcionais já existentes.
- Refatorações amplas fora de `bootstrap/domain`.

## Definitions

- `S5998`: regra Sonar que sinaliza repetições regex suscetíveis a stack overflow para entradas grandes.
- `Possessive quantifier`: quantificador regex (`++`, `*+`) que evita backtracking recursivo.
- `Input malformado grande`: string de versão com milhares de segmentos inválidos, como `"1.".repeat(n)`.

## Existing Context

- Issues abertos reportados:
- `6b818e1b-9460-4d3b-8293-1225c24cd0f9` (`RuntimeLibraryVersionComparator.java:10`)
- `9f768ff3-03a1-401b-bfee-d09fcfc9844e` (`RuntimeLibraryVersionComparator.java:124`)
- Evidência técnica já reproduzida localmente: padrões atuais estouram stack com entradas inválidas grandes (~2000 repetições com `-Xss1m`).
- `RuntimeLibraryIdentity` usa outro regex de parsing de arquivo JAR e será coberto preventivamente no mesmo esforço, conforme decisão de escopo.

## Desired End State

- `RuntimeLibraryVersionComparator` deixa de usar regex vulnerável a stack overflow mantendo a semântica atual.
- Testes unitários cobrem entradas extremas e impedem regressão do problema.
- `RuntimeLibraryIdentity` ganha proteção preventiva para rejeição rápida de nomes claramente inválidos muito grandes.
- `./mvnw clean verify` passa.
- Os dois issue keys acima não aparecem mais em `OPEN/CONFIRMED` na API Sonar.

## Milestones

### Milestone 1 - Reproduzir e travar regressão com testes (RED)

#### Goal

Codificar cenários extremos que demonstram o risco atual e devem permanecer verdes após o fix.

#### Changes

- [x] Editar `src/test/java/com/seed4j/cli/bootstrap/domain/RuntimeLibraryVersionComparatorTest.java`.
- [x] Adicionar teste com versão numérica inválida muito longa (ex.: `"1.".repeat(2500)`), verificando resultado `UNCOMPARABLE` sem lançar exceção.
- [x] Adicionar teste com versão qualificada inválida muito longa (ex.: `"1.".repeat(2500) + "1-"`), verificando `UNCOMPARABLE` sem lançar exceção.
- [x] Manter estilo Given/When/Then e assertions explícitas no corpo do teste.

#### Validation

- [x] Command: `./mvnw -Dtest=RuntimeLibraryVersionComparatorTest test`
- [x] Expected result: antes do fix, pelo menos um cenário novo falha por `StackOverflowError`; após o fix, todos os testes da classe passam.

#### Acceptance Criteria

- [x] Há cobertura explícita para os 2 formatos de entrada extrema.
- [x] O bug deixa de ser reproduzível com os novos testes.

### Milestone 2 - Corrigir regex vulnerável no comparador

#### Goal

Eliminar a causa raiz dos issues S5998 no comparador de versões sem mudar contrato de comparação.

#### Changes

- [x] Editar `src/main/java/com/seed4j/cli/bootstrap/domain/RuntimeLibraryVersionComparator.java`.
- [x] Substituir `QUALIFIED_NUMERIC_VERSION_PATTERN` por versão com quantificadores possessivos.
- [x] Introduzir `Pattern` pré-compilado para versão numérica simples e parar de usar `String.matches(...)`.
- [x] Manter comportamento atual para casos já cobertos: versões iguais, mais nova/mais antiga, sufixo `v`, qualificador igual/diferente, overflow de `Integer`.

#### Validation

- [x] Command: `./mvnw -Dtest=RuntimeLibraryVersionComparatorTest test`
- [x] Expected result: todos os testes de `RuntimeLibraryVersionComparatorTest` passam, incluindo os novos cenários extremos.

#### Acceptance Criteria

- [x] Não há mais uso de regex vulnerável nas duas ocorrências apontadas pelo Sonar.
- [x] Sem regressão de comportamento comparativo em casos existentes.

### Milestone 3 - Hardening preventivo no parsing de identidade por nome de JAR

#### Goal

Adicionar defesa leve adjacente para entradas inválidas muito grandes no parsing de `RuntimeLibraryIdentity`.

#### Changes

- [x] Editar `src/main/java/com/seed4j/cli/bootstrap/domain/RuntimeLibraryIdentity.java`.
- [x] Adicionar short-circuit de rejeição para nomes sem `-` ou sem sufixo `.jar` antes do regex (mesmo resultado funcional, menor custo).
- [x] Criar `src/test/java/com/seed4j/cli/bootstrap/domain/RuntimeLibraryIdentityTest.java`.
- [x] Cobrir parsing válido (`RELEASE`, `v1.2.3`) e inválido muito grande (retorna `Optional.empty()` sem exceção).
- [x] Garantir que testes existentes de seleção de libs continuam consistentes.

#### Validation

- [x] Command: `./mvnw -Dtest=RuntimeLibraryIdentityTest,RuntimeExtensionMissingLibrariesSelectorTest test`
- [x] Expected result: classes de teste passam; cenário inválido grande não lança exceção.

#### Acceptance Criteria

- [x] Parsing por nome de arquivo tem cobertura dedicada e explícita.
- [x] A proteção preventiva não altera resultados funcionais esperados.

### Milestone 4 - Validação completa e fechamento no Sonar

#### Goal

Comprovar estabilidade local e encerramento dos dois issues no Sonar.

#### Changes

- [ ] Executar validação completa do projeto.
- [ ] Executar análise Sonar local conforme `documentation/sonar.md`.
- [ ] Aguardar conclusão da CE task antes da consulta final de issues.
- [ ] Confirmar ausência dos dois issue keys alvo em `OPEN/CONFIRMED`.

#### Validation

- [ ] Command: `./mvnw clean verify`
- [ ] Expected result: build verde (testes, checkstyle, cobertura).
- [ ] Command: `docker compose -f src/main/docker/sonar.yml up -d`
- [ ] Expected result: Sonar disponível em `http://localhost:9001`.
- [ ] Command: `docker logs -f sonar-token && SONAR_TOKEN=$(docker logs sonar-token)`
- [ ] Expected result: token disponível em `SONAR_TOKEN`.
- [ ] Command: `./mvnw clean verify sonar:sonar -Dsonar.token=$SONAR_TOKEN`
- [ ] Expected result: análise enviada com sucesso.
- [ ] Command: `CE_TASK_URL=$(find . -name report-task.txt -exec grep '^ceTaskUrl=' {} \\; | head -n1 | cut -d= -f2-) && until curl -s "$CE_TASK_URL" | rg -q '"status":"SUCCESS"'; do sleep 2; done`
- [ ] Expected result: task Sonar concluída com `SUCCESS`.
- [ ] Command: `curl -s 'http://localhost:9001/api/issues/search?componentKeys=Seed4JCli&statuses=OPEN,CONFIRMED&impactSoftwareQualities=RELIABILITY&ps=200' | rg -n '6b818e1b-9460-4d3b-8293-1225c24cd0f9|9f768ff3-03a1-401b-bfee-d09fcfc9844e'`
- [ ] Expected result: sem saída (os dois issues não aparecem mais como abertos).

#### Acceptance Criteria

- [ ] `clean verify` aprovado.
- [ ] Os 2 issue keys específicos não aparecem em `OPEN/CONFIRMED`.

## Progress

- [x] Milestone 1 started
- [x] Milestone 1 completed
- [x] Milestone 2 started
- [x] Milestone 2 completed
- [x] Milestone 3 started
- [x] Milestone 3 completed
- [ ] Milestone 4 started
- [ ] Milestone 4 completed

## Decisions

- Decision: escopo inclui correção dos 2 issues S5998 + hardening preventivo adjacente em `RuntimeLibraryIdentity`.
  Rationale: escolha explícita do usuário para reduzir risco recorrente no mesmo fluxo de parsing.
  Date/Author: 2026-05-22 / Codex

- Decision: manter semântica de comparação de versão, alterando apenas mecanismo regex/validação.
  Rationale: reduzir risco de regressão funcional enquanto fecha bug de confiabilidade.
  Date/Author: 2026-05-22 / Codex

- Decision: gate final obrigatório é `clean verify` + confirmação via API Sonar.
  Rationale: build verde sozinho não prova fechamento de issue Sonar.
  Date/Author: 2026-05-22 / Codex

## Risks and Mitigations

- Risk: quantificadores possessivos rejeitarem algum formato hoje aceito.
  Mitigation: preservar regex equivalente em linguagem aceita e validar com suíte existente + cenários atuais de `RuntimeLibraryVersionComparatorTest`.

- Risk: short-circuit em `RuntimeLibraryIdentity` introduzir desvio funcional em nomes limítrofes.
  Mitigation: adicionar testes dedicados cobrindo válidos e inválidos conhecidos, além de rodar `RuntimeExtensionMissingLibrariesSelectorTest`.

- Risk: falso negativo na checagem Sonar por processamento assíncrono.
  Mitigation: aguardar CE task `SUCCESS` antes de consultar issues.

## Validation Strategy

1. Rodar testes focados imediatamente após cada milestone de código.
2. Rodar `./mvnw clean verify` como gate local final.
3. Rodar análise Sonar e esperar CE task concluir.
4. Confirmar fechamento por issue key na API Sonar.

## Rollout and Recovery

Rollout:

1. Commit único focado em confiabilidade do parsing de versão (`fix(bootstrap): ...`).
2. PR com evidências dos comandos de validação e resposta da API Sonar.

Recovery:

1. Se houver regressão funcional, reverter apenas mudanças de `RuntimeLibraryIdentity` e revalidar.
2. Se o problema estiver no comparador, reverter o commit e reaplicar em dois passos menores: testes primeiro, regex depois.

## Lessons Learned

- Os testes extremos reproduziram `StackOverflowError` em ambos os formatos malformados (`"1.".repeat(2500)` e `"1.".repeat(2500) + "1-"`) antes do fix, validando o risco de confiabilidade em runtime.
- A migração de `String.matches(...)` para `Pattern.matcher(...).matches()` com regex pré-compilada não alterou os resultados funcionais cobertos na suíte de comparação.
- O short-circuit em `RuntimeLibraryIdentity` (`.jar` + delimitador `-`) rejeita entrada inválida grande de forma imediata e manteve compatibilidade com testes de seleção de bibliotecas.
