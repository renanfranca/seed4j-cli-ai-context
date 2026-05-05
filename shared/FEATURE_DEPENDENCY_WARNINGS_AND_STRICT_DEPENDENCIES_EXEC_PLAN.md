# Tornar explicitas dependencias obrigatorias com avisos de risco e modo estrito opcional

Este ExecPlan e um documento vivo. Atualize `Progress`, `Decisions`, `Risks` e `Lessons Learned` durante a execucao.

## Purpose / Big Picture

Hoje o CLI mostra dependencias no `seed4j list`, mas nao deixa claro o suficiente que elas sao obrigatorias para runtime e que nao sao aplicadas automaticamente. Isso permite fluxo com sucesso parcial: o modulo pode ser aplicado e o projeto ficar em estado potencialmente quebrado.
Este plano adiciona mensagens explicitas de risco antes e depois do `apply`, mantem o comportamento padrao nao bloqueante e introduz um modo opcional `--strict-dependencies` para CI falhar cedo quando houver dependencia faltante.

## Scope

Escopo:

- Ajustar UX textual do `seed4j list` para reforcar semantica de dependencia obrigatoria/manual.
- Adicionar aviso pre-`apply` quando o modulo alvo tiver dependencias faltantes.
- Adicionar aviso pos-`apply` quando ainda existirem dependencias faltantes.
- Adicionar opcao `--strict-dependencies` para falha opcional em pipelines CI.
- Atualizar mensagens de ajuda (`-h`) para deixar o contrato explicito.
- Cobrir o comportamento com testes automatizados e atualizar documentacao de comandos.

Fora de escopo:

- Aplicar dependencias automaticamente em cascata.
- Resolver dependencia por grafo completo com ordenacao topologica.
- Alterar o contrato de `Seed4JModulesApplicationService` em bibliotecas externas.
- Alterar semantica de dependencias `feature:<slug>` alem da exibicao/mensagem.

## Definitions

- `Dependencia obrigatoria`: dependencia declarada em `organization.dependencies()` de um modulo e necessaria para comportamento correto do modulo aplicado.
- `Dependencia faltante`: dependencia obrigatoria que ainda nao esta aplicada no projeto alvo.
- `Modo padrao`: execucao sem `--strict-dependencies`; apenas avisos (`WARN`), sem bloqueio.
- `Modo estrito`: execucao com `--strict-dependencies`; falta de dependencia interrompe aplicacao com exit code nao-zero.
- `Contrato manual`: dependencias nao sao auto-aplicadas por design da CLI.

## Existing Context

- `list` hoje renderiza coluna de dependencias com tokens tipados `module:<slug>` e `feature:<slug>` em [ListModulesCommand.java](/home/renanfranca/projects/seed4j-cli/src/main/java/com/seed4j/cli/command/infrastructure/primary/ListModulesCommand.java).
- `apply` e modelado como comando com subcomandos por slug de modulo em [ApplyModuleCommand.java](/home/renanfranca/projects/seed4j-cli/src/main/java/com/seed4j/cli/command/infrastructure/primary/ApplyModuleCommand.java) e [ApplyModuleSubCommand.java](/home/renanfranca/projects/seed4j-cli/src/main/java/com/seed4j/cli/command/infrastructure/primary/ApplyModuleSubCommand.java).
- O fluxo atual de `ApplyModuleSubCommand#call` aplica diretamente `modules.apply(...)` sem pre-check ou pos-check de dependencia faltante.
- Ja existe exemplo testado de acesso ao historico do projeto via `ProjectsApplicationService` em [Seed4JCommandsFactoryTest.java](/home/renanfranca/projects/seed4j-cli/src/test/java/com/seed4j/cli/command/infrastructure/primary/Seed4JCommandsFactoryTest.java):

```java
ProjectHistory history = projects.getHistory(new ProjectPath(projectPath.toString()));
Object value = history.latestProperties().parameters().getOrDefault(propertyKey, null);
```

- O contrato atual em documentacao descreve a coluna `Dependencies`, mas nao explicita risco operacional de nao aplicar dependencias em [Commands.md](/home/renanfranca/projects/seed4j-cli/documentation/Commands.md).

## Desired End State

- `seed4j list` continua informativo, mas reforca explicitamente o contrato manual:
  - `Dependencies: required for module runtime, applied manually by design.`
- Antes do `apply`, quando houver dependencia faltante, a CLI mostra aviso claro sem bloquear no modo padrao:
  - `[WARN] Module \`<module-slug>\` requires: <dependency-list>`
  - `[WARN] These dependencies are NOT auto-applied (CLI design).`
  - `[RISK] Module may be generated, but may not compile/run correctly.`
  - `[NEXT] Suggested: seed4j apply <dependency-slug> ...`
- Depois de `apply` bem-sucedido, se ainda houver pendencia:
  - `[DONE] Module \`<module-slug>\` applied.`
  - `[WARN] Required dependencies still missing: <dependency-list>`
  - `[WARN] Project is in a potentially non-functional state until dependencies are applied.`
- Ajuda (`-h`) de comandos relevantes inclui frase curta:
  - `Dependencies are required for correct behavior, but must be applied manually.`
- `--strict-dependencies` fica disponivel como modo opcional para CI e falha cedo quando existir dependencia faltante.

## Milestones

### Milestone 1 - Modelar deteccao de dependencias faltantes

#### Goal

Criar tipos dedicados e uma estrategia deterministica para descobrir dependencias faltantes antes/depois do `apply` sem acoplar regra de negocio em output.

#### Changes

- [ ] Introduzir tipos de dominio no contexto de comando para representar `DependencyRequirement`, `MissingDependencies` e `DependencyCheckResult` (nomes finais podem variar, mantendo Types Driven Development).
- [ ] Implementar resolvedor de pendencias para modulo alvo usando `Seed4JModuleResource.organization().dependencies()` e estado atual do projeto.
- [ ] Usar explicitamente o acesso de historico ja validado no projeto:
  - `ProjectHistory history = projects.getHistory(new ProjectPath(projectPath.toString()));`
  - `history.latestProperties().parameters()`
- [ ] Definir como extrair deste historico os sinais de dependencias/modulos ja aplicados (sem inventar regra fora dos dados reais).
- [ ] Definir regra de escopo: incluir apenas dependencias do tipo `MODULE` no bloqueio estrito; dependencias `FEATURE` permanecem informativas (registrar decisao).
- [ ] Garantir ordenacao estavel da lista de dependencias para mensagens e testes deterministicos.

#### Validation

- [ ] Command: `./mvnw -Dtest=ListModulesCommandTest,Seed4JCommandsFactoryTest test`
- [ ] Expected result: suite segue verde com novas classes compilando e sem regressao de comportamento existente.

#### Acceptance Criteria

- [ ] Existe uma API interna clara para avaliar pendencias de dependencia, reutilizavel no pre e pos-`apply`.
- [ ] O resultado do check diferencia explicitamente: sem pendencia, com pendencia e modo estrito.

### Milestone 2 - Reforcar semantica em list e help

#### Goal

Fazer o usuario enxergar o contrato manual de dependencias antes de executar `apply`.

#### Changes

- [ ] Atualizar descricao de `list` e/ou linha informativa no output em [ListModulesCommand.java](/home/renanfranca/projects/seed4j-cli/src/main/java/com/seed4j/cli/command/infrastructure/primary/ListModulesCommand.java) para incluir: `Dependencies: required for module runtime, applied manually by design.`
- [ ] Atualizar textos de ajuda em [ApplyModuleCommand.java](/home/renanfranca/projects/seed4j-cli/src/main/java/com/seed4j/cli/command/infrastructure/primary/ApplyModuleCommand.java) e (quando aplicavel) em subcomandos para incluir: `Dependencies are required for correct behavior, but must be applied manually.`
- [ ] Ajustar documentacao em [Commands.md](/home/renanfranca/projects/seed4j-cli/documentation/Commands.md) para refletir a nova semantica de aviso.

#### Validation

- [ ] Command: `./mvnw -Dtest=ListModulesCommandTest,Seed4JCommandsFactoryTest test`
- [ ] Expected result: asserts de help/list verificam o novo texto sem quebrar formato de tabela.

#### Acceptance Criteria

- [ ] `seed4j list` deixa explicito o contrato manual de dependencias.
- [ ] `seed4j apply --help` (e ajuda relevante) comunica o mesmo contrato.

### Milestone 3 - Aviso pre-apply e modo estrito opcional

#### Goal

Avisar risco antes da aplicacao e permitir bloqueio opcional em CI.

#### Changes

- [ ] Adicionar opcao `--strict-dependencies` no escopo de `apply <module-slug>` em [ApplyModuleSubCommand.java](/home/renanfranca/projects/seed4j-cli/src/main/java/com/seed4j/cli/command/infrastructure/primary/ApplyModuleSubCommand.java).
- [ ] Antes de `modules.apply(...)`, calcular dependencias faltantes do modulo alvo.
- [ ] No modo padrao, imprimir bloco `[WARN]/[RISK]/[NEXT]` e seguir.
- [ ] No modo estrito, imprimir aviso e retornar exit code nao-zero sem executar `modules.apply(...)`.
- [ ] Definir mensagem de sugestao `seed4j apply <dependency> ...` para uma ou multiplas dependencias mantendo ordem previsivel.

#### Validation

- [ ] Command: `./mvnw -Dtest=Seed4JCommandsFactoryTest test`
- [ ] Expected result: novos testes cobrem pre-aviso e falha em modo estrito sem efeitos colaterais (sem commit/sem alteracao de historico).

#### Acceptance Criteria

- [ ] Sem `--strict-dependencies`: dependencia faltante nao bloqueia, mas aviso aparece sempre antes do `apply`.
- [ ] Com `--strict-dependencies`: dependencia faltante bloqueia e retorna exit code nao-zero.

### Milestone 4 - Aviso pos-apply com pendencia residual

#### Goal

Confirmar sucesso da aplicacao sem esconder divida tecnica restante.

#### Changes

- [ ] Apos `modules.apply(...)` bem-sucedido, recalcular pendencias e emitir bloco `[DONE]/[WARN]` quando ainda faltar dependencia.
- [ ] Evitar ruido: quando nao houver pendencia residual, nao imprimir bloco de risco.
- [ ] Garantir que mensagens pos-apply nao conflitam com erros de parametros obrigatorios ou falhas internas ja existentes.

#### Validation

- [ ] Command: `./mvnw -Dtest=Seed4JCommandsFactoryTest test`
- [ ] Expected result: cenario feliz com pendencia residual mostra `[DONE]` seguido de avisos; cenario sem pendencia nao mostra avisos.

#### Acceptance Criteria

- [ ] Usuario recebe confirmacao de aplicacao e estado de risco residual de forma objetiva.
- [ ] Fluxo atual de validacao de parametros obrigatorios permanece intacto.

### Milestone 5 - Fechamento de documentacao e validacao completa

#### Goal

Finalizar contrato funcional e validar regressao completa do repositorio.

#### Changes

- [ ] Atualizar exemplos de `list`, `apply` e `--strict-dependencies` em [Commands.md](/home/renanfranca/projects/seed4j-cli/documentation/Commands.md).
- [ ] Incluir exemplo de uso CI com falha intencional quando faltar dependencia.
- [ ] Revisar formato e consistencia de mensagens com o padrao textual aprovado (`WARN`, `RISK`, `NEXT`, `DONE`).

#### Validation

- [ ] Command: `./mvnw clean verify`
- [ ] Expected result: testes unitarios/integracao, checkstyle e cobertura verdes.
- [ ] Command: `npm run prettier:check`
- [ ] Expected result: sem divergencia de formatacao.

#### Acceptance Criteria

- [ ] Contrato final esta documentado com exemplos executaveis.
- [ ] Pipeline local completo passa sem regressao.

## Progress

- [ ] Milestone 1 started
- [ ] Milestone 1 completed
- [ ] Milestone 2 started
- [ ] Milestone 2 completed
- [ ] Milestone 3 started
- [ ] Milestone 3 completed
- [ ] Milestone 4 started
- [ ] Milestone 4 completed
- [ ] Milestone 5 started
- [ ] Milestone 5 completed

## Decisions

- Decision: comportamento padrao permanece nao bloqueante (somente avisos).
  Rationale: preserva UX atual e evita quebra inesperada para usuarios locais.
  Date/Author: 2026-05-05 / User + Codex

- Decision: `--strict-dependencies` sera opcional e voltado a CI, com falha antes de aplicar modulo quando houver pendencia.
  Rationale: oferece governanca em pipeline sem enrijecer o fluxo padrao.
  Date/Author: 2026-05-05 / User + Codex

- Decision: mensagens de risco devem ser objetivas, padronizadas e com proximo passo explicito.
  Rationale: evitar falso sentimento de sucesso apos `apply` parcial.
  Date/Author: 2026-05-05 / User + Codex

## Risks and Mitigations

- Risk: API de historico disponivel pode nao expor diretamente lista de modulos aplicados.
  Mitigation: iniciar pelo acesso comprovado via `projects.getHistory(new ProjectPath(projectPath.toString()))` e `history.latestProperties().parameters()`; decidir regra final somente apos validar os dados reais presentes no historico.

- Risk: dependencias do tipo `FEATURE` podem nao ter correspondencia direta com comando `seed4j apply <slug>`.
  Mitigation: restringir bloqueio estrito a dependencias `MODULE` e manter `FEATURE` apenas como aviso informativo, registrando decisao explicitamente.

- Risk: novas mensagens podem quebrar testes sensiveis a output textual e regex de tabela.
  Mitigation: atualizar testes com asserts focados por linha e preservar estrutura tabular atual.

- Risk: escolha inadequada de stream (`stdout` vs `stderr`) pode atrapalhar parsing em automacoes.
  Mitigation: padronizar stream por tipo de mensagem e documentar no `Commands.md`; cobrir com testes de captura.

- Risk: `--strict-dependencies` mal posicionado na arvore de comandos pode gerar UX confusa.
  Mitigation: documentar sintaxe oficial (`seed4j apply <module> --strict-dependencies`) e validar help/output com testes.

## Validation Strategy

1. Rodar testes focados em `list` e `apply` para feedback rapido.
2. Rodar `./mvnw clean verify` para garantir cobertura/checkstyle/integracao.
3. Rodar `npm run prettier:check` para garantir formato.
4. Validar manualmente cenarios CLI:
   - `seed4j list`
   - `seed4j apply <module-com-dependencia>`
   - `seed4j apply <module-com-dependencia> --strict-dependencies`

## Rollout and Recovery

- Rollout: liberar sem habilitacao obrigatoria; comportamento padrao permanece compativel e `--strict-dependencies` entra como opcional.
- Recovery: em caso de ruido inesperado ou regressao de UX, reverter apenas o bloco de mensagens/flag mantendo melhorias de testes e documentacao isoladas por commit.

## Lessons Learned

- A preencher durante execucao.
