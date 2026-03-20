# Reduzir dívida técnica de build (convergência Maven + aviso do Mockito)

Este ExecPlan é um documento vivo. Atualize `Progress`, `Decisions`, `Risks` e `Lessons Learned` durante a execução.

## Purpose / Big Picture

Hoje o build passa, mas com alertas que escondem risco real: conflitos de versões transitivas e auto-attach do Mockito em JDK 25. Isso aumenta ruído, dificulta detectar regressões e pode quebrar em upgrade futuro do JDK ou em endurecimento de regras CI. O objetivo é chegar em `./mvnw clean verify` sem esses alertas, com configuração explícita e documentada.

## Scope

Escopo:

- Resolver os conflitos reportados por `DependencyConvergence` no Maven Enforcer.
- Eliminar o aviso de self-attach do Mockito, configurando agente Java no pipeline de testes.
- Documentar as decisões técnicas e atualizar status de dívida técnica.

Fora de escopo:

- Refatorar código funcional da CLI.
- Alterar arquitetura hexagonal.
- Atualizar dependências além do necessário para convergência e estabilidade dos testes.

## Definitions

- `Dependency Convergence`: regra do Maven Enforcer que exige uma única versão final por coordenada (`groupId:artifactId`) no grafo de dependências.
- `Transitiva`: dependência trazida por outra dependência (não declarada diretamente no `pom.xml` do projeto).
- `Inline mock maker`: mecanismo do Mockito que usa instrumentação para mockar tipos finais/estáticos; em JDK moderno, exige agente Java explícito.
- `Java agent`: JAR passado via `-javaagent:` na JVM de teste para instrumentação em startup.
- `argLine`: parâmetro do Surefire/Failsafe com flags JVM; JaCoCo também injeta opções nele, então composição incorreta pode quebrar cobertura.

## Existing Context

- Arquivo principal de build: `pom.xml`.
- O Enforcer já possui execução `enforce-dependencyConvergence`, mas com `<fail>false</fail>`, o que mantém build verde com warnings.
- Baseline coletado em 2026-03-20 com `./mvnw -DskipTests enforcer:enforce@enforce-dependencyConvergence` mostrou conflitos para:
  - `org.checkerframework:checker-qual` (`3.37.0` vs `3.52.0`)
  - `org.codehaus.plexus:plexus-utils` (`3.5.1` vs `3.2.0`)
  - `com.google.errorprone:error_prone_annotations` (`2.21.1` vs `2.41.0`)
  - `org.apache.commons:commons-text` (`1.15.0` vs `1.13.1`)
- O item `commons-io:commons-io` listado em `shared/TECH_DEBT.md` não apareceu nesse baseline; precisa ser revalidado durante execução.
- `maven-surefire-plugin` e `maven-failsafe-plugin` não possuem `argLine` explícito hoje.
- Não há configuração explícita de mock maker em `src/test/resources/mockito-extensions`.
- Referência upstream validada: o PR `seed4j/seed4j#13199` (merged em 2025-09-17) resolveu o mesmo warning adicionando:
  - `maven-dependency-plugin` com goal `properties`;
  - `argLine` em Surefire/Failsafe com `@{argLine} -javaagent:${org.mockito:mockito-core:jar}`.

## Desired End State

- `DependencyConvergence` sem warnings para os artefatos monitorados.
- Decisão explícita sobre política:
  - preferencial: ativar convergência estrita (`<fail>true</fail>`) após zerar conflitos;
  - alternativa aceita: manter `fail=false` somente com exceções justificadas e documentadas.
- Execução de testes (`test` e `verify`) sem aviso de self-attach do Mockito.
- `TECH_DEBT.md` atualizado com status final e evidência de validação.

## Milestones

### Milestone 1 - Baseline completo e decisão de política

#### Goal

Consolidar diagnóstico técnico (conflitos reais e origem) e decidir critério de aceitação para convergência.

#### Changes

- [x] Executar `dependency:tree` com filtro por artefatos conflitantes para confirmar caminhos de resolução.
- [x] Registrar no `shared/TECH_DEBT.md` os conflitos efetivos observados na data da execução (inclusive se `commons-io` não reproduzir).
- [x] Definir e registrar em `Decisions` se o alvo é convergência estrita (`fail=true`) nesta entrega.

#### Validation

- [x] Command: `./mvnw -DskipTests enforcer:enforce@enforce-dependencyConvergence`
- [x] Expected result: lista de conflitos inicial reproduzida e documentada.
- [x] Command: `./mvnw -DskipTests dependency:tree -Dverbose -Dincludes=org.checkerframework:checker-qual,org.codehaus.plexus:plexus-utils,com.google.errorprone:error_prone_annotations,commons-io:commons-io,org.apache.commons:commons-text`
- [x] Expected result: caminhos de cada artefato conflitante identificados para guiar overrides/exclusions.

#### Acceptance Criteria

- [x] Existe baseline reproduzível com comandos e saída esperada.
- [x] A política de convergência (estrita ou exceção documentada) foi decidida antes de editar versões.

### Milestone 2 - Resolver convergência de dependências

#### Goal

Eliminar conflitos de versões transitivas sem quebrar comportamento de runtime/testes.

#### Changes

- [x] Editar `pom.xml` para alinhar versões via `dependencyManagement` (ou exclusões pontuais justificadas) para os artefatos conflitantes.
- [x] Priorizar alinhamento com ecossistema dominante do grafo (Spring Boot/seed4j) para reduzir risco de incompatibilidade.
- [x] Se necessário, adicionar comentário curto no `pom.xml` explicando por que o override existe e quando remover. (não foi necessário nesta entrega)
- [x] Decidir entre:
  - [x] Ativar `<fail>true</fail>` no `enforce-dependencyConvergence`; ou
  - [ ] Manter `fail=false` com justificativa explícita dos remanescentes no `TECH_DEBT.md`.

#### Validation

- [x] Command: `./mvnw -DskipTests enforcer:enforce@enforce-dependencyConvergence`
- [x] Expected result: sem warnings de convergência para itens tratados.
- [x] Command: `./mvnw -DskipTests dependency:tree -Dverbose -Dincludes=org.checkerframework:checker-qual,org.codehaus.plexus:plexus-utils,com.google.errorprone:error_prone_annotations,commons-io:commons-io,org.apache.commons:commons-text`
- [x] Expected result: cada coordenada converge para versão única (ou exceção documentada).

#### Acceptance Criteria

- [x] Não há conflito ativo sem decisão explícita.
- [x] Build continua verde após alinhamento de versões.

### Milestone 3 - Remover aviso de self-attach do Mockito

#### Goal

Garantir execução de testes compatível com futuros JDKs, com agente Java configurado de forma explícita.

#### Changes

- [x] Editar `pom.xml` para adicionar `argLine` em Surefire e Failsafe preservando flags de JaCoCo (`@{...}` ou propriedade dedicada).
- [x] Adicionar `maven-dependency-plugin` (versão em `properties`) com execução `generate-dependencies-properties` e goal `properties`.
- [x] Configurar agente Mockito usando o mesmo padrão do PR `seed4j/seed4j#13199`: `-javaagent:${org.mockito:mockito-core:jar}`.
- [x] Aplicar em ambos os plugins de teste:
  - [x] Surefire: `<argLine>@{argLine} -javaagent:${org.mockito:mockito-core:jar}</argLine>`
  - [x] Failsafe: `<argLine>@{argLine} -javaagent:${org.mockito:mockito-core:jar}</argLine>`
- [x] Confirmar se a configuração vale para unit e integration tests (Surefire + Failsafe).
- [x] Evitar sobrescrever o `argLine` injetado pelo JaCoCo (risco de perda de cobertura).
- [x] Adicionar fallback `<argLine></argLine>` para evitar quebra de late substitution quando `argLine` não for previamente populado.

#### Validation

- [x] Command: `./mvnw test`
- [x] Expected result: testes passam sem mensagem de self-attach do Mockito.
- [x] Command: `./mvnw -DskipTests help:effective-pom`
- [x] Expected result: `argLine` efetivo de Surefire/Failsafe inclui `@{argLine}` e `-javaagent:${org.mockito:mockito-core:jar}`.
- [x] Command: `./mvnw -Dit.test=ExtensionRuntimeBootstrapPackagedJarIT,ExtensionRuntimeBootstrapListPackagedJarIT dependency:properties failsafe:integration-test failsafe:verify`
- [x] Expected result: integration tests relevantes passam sem aviso do Mockito.

#### Acceptance Criteria

- [x] Nenhum aviso de self-attach em logs de teste.
- [x] Cobertura JaCoCo continua sendo produzida (arquivos `.exec` e relatório em `target/jacoco/`).

### Milestone 4 - Fechamento de dívida e validação final

#### Goal

Concluir com documentação atualizada e validação fim-a-fim do repositório.

#### Changes

- [x] Atualizar `_temporary/ai_agent/seed4j-cli-ai-context/shared/TECH_DEBT.md` com status final dos itens 1 e 2.
- [x] Se necessário, atualizar `README.md` com nota curta sobre configuração de testes (Mockito agent) para evitar regressão em upgrades JDK. (não necessário nesta entrega)
- [x] Registrar decisões finais e aprendizados neste ExecPlan.

#### Validation

- [x] Command: `./mvnw clean verify`
- [x] Expected result: build completo verde, sem warnings tratados nesta dívida.
- [x] Command: `npm run prettier:check`
- [x] Expected result: sem divergência de formatação.

#### Acceptance Criteria

- [x] Débitos tratados aparecem como concluídos ou com exceção formalizada.
- [x] Evidência de validação completa registrada.

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

- Decision: Resolver convergência antes de endurecer a regra para `fail=true`.
  Rationale: evita transformar diagnóstico em bloqueio total sem caminho de correção validado.
  Date/Author: 2026-03-20 / Codex

- Decision: Ativar convergência estrita (`<fail>true</fail>`) nesta entrega.
  Rationale: os conflitos foram zerados e isso reduz risco de regressão silenciosa no classpath.
  Date/Author: 2026-03-20 / Codex

- Decision: Tratar `commons-io:commons-io` como duplicado sem conflito e remover do conjunto de conflitos ativos.
  Rationale: revalidação em `dependency:tree -Dverbose` mostrou apenas duplicate, sem version clash.
  Date/Author: 2026-03-20 / Codex

- Decision: Reusar a estratégia do PR `seed4j/seed4j#13199` para Mockito agent.
  Rationale: padrão já validado e mergeado upstream; evita hardcode manual de caminho/versão do JAR.
  Date/Author: 2026-03-20 / Codex

- Decision: Adicionar fallback `<argLine></argLine>` no `pom.xml`.
  Rationale: evita falha de fork por placeholder `@{argLine}` literal quando execução não popular a propriedade.
  Date/Author: 2026-03-20 / Codex

## Risks and Mitigations

- Risk: Forçar versão transitive incompatível com `seed4j` ou `mongock`.
  Mitigation: validar via `dependency:tree`, testes direcionados e `clean verify` completo.

- Risk: Sobrescrever `argLine` e quebrar instrumentação JaCoCo.
  Mitigation: compor `argLine` explicitamente preservando parâmetros injetados pelo JaCoCo e validar geração de relatórios.

- Risk: Placeholder `${org.mockito:mockito-core:jar}` não resolver sem `maven-dependency-plugin:properties`.
  Mitigation: adicionar execução `generate-dependencies-properties`, confirmar via `help:effective-pom` e usar lifecycle (`verify`) ou `dependency:properties` ao executar failsafe goals diretamente.

- Risk: Resolver warnings locais e deixar comportamento frágil para próximos upgrades.
  Mitigation: documentar decisão (inclusive exceções) no `TECH_DEBT.md` e, se aplicável, no `README.md`.

- Risk: Build ficar verde apenas porque Enforcer permanece em modo permissivo.
  Mitigation: registrar decisão formal; se mantiver `fail=false`, listar remanescentes com prazo e responsável.

## Validation Strategy

1. Reproduzir baseline dos warnings (`enforcer` + `dependency:tree`).
2. Aplicar alinhamentos de versão e validar convergência sem warnings.
3. Ajustar agente Mockito em Surefire/Failsafe preservando JaCoCo.
4. Rodar testes unitários e integrações críticas.
5. Rodar `./mvnw clean verify` e `npm run prettier:check`.

## Rollout and Recovery

Rollout:

1. Entregar mudanças em commit único focado em build/test runtime (`build(pom)` ou `chore(build)`).
2. Validar pipeline CI com os mesmos comandos locais.

Recovery:

1. Se houver regressão de runtime/testes, reverter apenas overrides adicionados no `dependencyManagement`.
2. Se houver falha de cobertura, remover temporariamente ajuste de agente Mockito e restaurar `argLine` anterior enquanto se corrige composição com JaCoCo.
3. Manter registro no `TECH_DEBT.md` com status real após rollback parcial.

## Lessons Learned

- `commons-io` aparecia como hipótese de conflito, mas a evidência atual mostrou duplicação sem divergência de versão; revalidação evita correção desnecessária.
- Convergência estrita (`fail=true`) só foi ativada depois de confirmar baseline, caminhos transitivos e validação pós-fix.
- `@{argLine}` e `${org.mockito:mockito-core:jar}` funcionam bem no lifecycle completo, mas execução direta de goals precisa garantir que `dependency:properties` rode no mesmo comando.
- O fallback `<argLine></argLine>` evita falha de inicialização da JVM quando a propriedade não foi preenchida.
