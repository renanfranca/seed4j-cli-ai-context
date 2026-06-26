# Validar Java 25+ no launcher npm

This ExecPlan is a living document. Keep `Progress`, `Decisions`, `Risks`, and `Lessons Learned` up to date as work advances.

## Purpose / Big Picture

Antes de iniciar o JAR, o launcher npm verificará `java --version`. Java ausente, anterior à versão 25 ou com saída não reconhecida falhará de forma controlada, sem expor `UnsupportedClassVersionError`.

## Scope

Incluído:

- Validação de Java 25 ou superior no primeiro `java` do `PATH`.
- Mensagens prescritivas e exit code `1` para falhas de validação.
- Preservação do contrato atual de argumentos, stdio, sinais e exit code do JAR.
- Documentação do requisito do CLI.

Excluído:

- Download de JVM, jDeploy, runtime privado ou alterações no projeto gerado.
- Alterações no código Java, `pom.xml` ou workflows de release.

Safety boundary: This task is limited to authorized, defensive maintenance of this repository.

## Definitions

- **Launcher npm:** `bin/seed4j.js`.
- **Java compatível:** versão principal maior ou igual a 25.
- **Saída reconhecida:** linha iniciada por `openjdk <major>` ou `java <major>`.

## Existing Context

- O launcher atualmente executa diretamente `java -jar`.
- O teste público está em `test/npm/seed4j-command.test.js`.
- O README informa Java 25, mas não “25 ou superior” nem distingue o requisito do CLI daquele dos projetos gerados.
- `npm run test:npm-package` passa no estado inicial.
- O `pom.xml` já define Java 25 como mínimo.

## Desired End State

`bin/seed4j.js` executará `java --version` de forma síncrona, capturando stdout e stderr sem consumir o stdin do usuário. Somente após validação bem-sucedida iniciará o JAR com o bloco de `spawn` atual.

Contratos de erro:

- Java ausente: informar que Java 25 ou superior deve estar no `PATH`.
- Java abaixo de 25: informar a versão encontrada e orientar a instalar uma versão compatível que seja o primeiro `java` do `PATH`.
- Saída inválida ou comando malsucedido: informar que não foi possível determinar a versão por `java --version`.
- Todas essas falhas encerram com código `1` e não iniciam o JAR.

Não haverá nova API pública; apenas o comportamento de inicialização e diagnóstico do comando `seed4j` mudará.

## Milestones

### Milestone 1 — Registrar o ExecPlan e preparar testes comportamentais

#### Goal

Registrar este plano e preparar a suíte pública para observar separadamente a consulta de versão e a execução do JAR, incluindo contratos de stdio e sinal.

#### Changes

- Salvar este plano em `/home/renanfranca/projects/seed4j-cli/_temporary/ai_agent/seed4j-cli-ai-context/shared/2026-06-25_FEATURE_validate-java-version-in-npm-launcher-exec-plan.md`.
- Adaptar o Java falso em `test/npm/seed4j-command.test.js` para responder separadamente a `--version` e `-jar`.
- Manter todos os testes na suíte pública existente, sem criar testes para helpers internos.
- Adicionar caracterizações para stdio e `SIGTERM`.

#### Validation

- Command: `npm run test:npm-package`
- Expected result: a suíte pública passa antes da introdução de cada novo comportamento ou falha pelo motivo previsto no ciclo RED.

#### Acceptance Criteria

- A suíte observa o launcher exclusivamente pelo comando público.
- Os contratos existentes de argumentos, stdio, sinal e exit code estão cobertos.

### Milestone 2 — Implementar por TDD a validação

#### Goal

Recusar ambientes Java incompatíveis antes de iniciar o JAR e preservar o fluxo atual quando a versão for compatível.

#### Changes

- Executar ciclos independentes RED-GREEN-REFACTOR para Java ausente, Java 21, saída não reconhecida e Java 25/26.
- Em `bin/seed4j.js`, definir versão mínima `25`, executar `java --version` com `spawnSync`, combinar stdout e stderr e extrair o major com correspondência multiline para `openjdk` ou `java`.
- Manter o parser como detalhe interno do launcher.
- Preservar sem alterações semânticas o `spawn('java', ['-jar', ...])` e seus handlers.

#### Validation

- Command after each cycle: `npm run test:npm-package`
- Expected result: o teste novo falha antes da implementação pelo motivo previsto e a suíte completa passa após a mudança mínima.

#### Acceptance Criteria

- Java ausente, Java 21, saída inválida e comando malsucedido encerram com código `1` sem iniciar o JAR.
- Java 25 e Java 26 iniciam o JAR normalmente.

### Milestone 3 — Preservar contratos e documentar

#### Goal

Confirmar a compatibilidade comportamental do launcher e esclarecer o requisito Java para usuários do CLI.

#### Changes

- Confirmar argumentos, exit code, stdin, stdout, stderr e sinal do processo Java em `test/npm/seed4j-command.test.js`.
- Atualizar o Quick Start e a seção Java do `README.md` para “Java 25 ou superior”.
- Esclarecer que Java é requisito de execução do `seed4j-cli`; projetos gerados, inclusive frontend-only, possuem requisitos próprios.

#### Validation

- Command: `npm run test:npm-package`
- Command: `npm run prettier:check`
- Command: `./mvnw --batch-mode -ntp -DskipTests package`
- Command: `npm run package:prepare`
- Command: `node bin/seed4j.js --version`
- Expected result: todos os comandos encerram com código `0`, e o smoke test imprime a versão do CLI empacotado.

#### Acceptance Criteria

- Todos os contratos públicos do launcher estão cobertos e passam.
- O README descreve Java 25+ como requisito do CLI, separado dos requisitos dos projetos gerados.

## Progress

- [x] ExecPlan salvo
- [x] Milestone 1 iniciado
- [x] Milestone 1 concluído
- [x] Milestone 2 iniciado
- [x] Milestone 2 concluído
- [x] Milestone 3 iniciado
- [x] Milestone 3 concluído
- [x] Validações focadas concluídas
- [x] Usuário solicitado a executar `./mvnw clean verify`

## Decisions

- Decision: aceitar qualquer versão principal `>= 25`.
  Rationale: acompanha o requisito mínimo do Maven sem rejeitar versões futuras.
  Date/Author: 2026-06-25 / Codex

- Decision: consultar exatamente `java --version`.
  Rationale: segue o contrato solicitado e usa o primeiro executável disponível no `PATH`.
  Date/Author: 2026-06-25 / Codex

- Decision: testar exclusivamente pelo comando público.
  Rationale: o parser não constitui API estável e não deve direcionar a estrutura da implementação.
  Date/Author: 2026-06-25 / Codex

- Decision: tratar exit code não zero de `java --version` como falha mesmo quando a saída contém uma versão reconhecível.
  Rationale: um comando malsucedido não fornece uma validação confiável do runtime e o contrato exige falha controlada.
  Date/Author: 2026-06-25 / Codex

## Risks and Mitigations

- Risk: a saída varia entre distribuições Java.
  Mitigation: aceitar prefixos `openjdk` e `java`, lendo stdout e stderr.
- Risk: a validação pode consumir stdin.
  Mitigation: executar `--version` com stdin ignorado e manter `stdio: 'inherit'` somente para o JAR.
- Risk: a pré-validação pode alterar sinais ou exit code.
  Mitigation: manter o bloco atual de execução do JAR e cobri-lo com testes comportamentais.
- Risk: o requisito fica duplicado entre Maven, launcher e documentação.
  Mitigation: manter `25` explícito e coberto nos testes, sem tentar ler `pom.xml` do pacote npm, onde ele não é distribuído.

## Validation Strategy

Cada novo comportamento será introduzido por um teste inicialmente falho. A suíte npm completa será executada em todos os ciclos. O caminho real com Java 25 e o JAR empacotado será validado ao final. O gate `./mvnw clean verify` será solicitado ao usuário, conforme as regras do repositório.

Resultados em 2026-06-25:

- `npm run test:npm-package`: passou.
- `npx prettier --check README.md bin/seed4j.js test/npm/seed4j-command.test.js`: passou.
- `npm run prettier:check`: falhou apenas por `.mvn/settings-no-mirror.xml`, arquivo preexistente e sem diff; o teste npm foi formatado e todos os arquivos alterados passaram no check focado.
- `./mvnw --batch-mode -ntp -DskipTests package`: passou.
- `npm run package:prepare`: passou.
- `node bin/seed4j.js --version`: passou e exibiu Seed4J CLI `0.0.3-SNAPSHOT`, Seed4J `2.2.0` e runtime mode `standard`.
- `git diff --check`: passou.

## Rollout and Recovery

A mudança seguirá o processo npm existente, sem migração ou alteração no workflow. Em caso de regressão, a recuperação consiste em reverter launcher, testes e documentação no mesmo commit.

## Lessons Learned

- O launcher já preserva argumentos, stdio, sinais do filho e exit code; a validação deve envolver esse fluxo, não reimplementá-lo.
- A suíte npm existente é o ponto adequado para todos os cenários.
- A documentação atual precisa esclarecer tanto `25+` quanto a separação entre requisitos do CLI e do projeto gerado.
- A caracterização de stdin por pipe através de um processo Node intermediário sofre uma corrida de EOF no harness; um descritor de arquivo testa de forma determinística o mesmo contrato `stdio: 'inherit'`.
- O check global de Prettier já encontra `.mvn/settings-no-mirror.xml` fora do padrão, sem relação com esta mudança; os arquivos alterados passam no check focado.
- `npm ci` emitiu aviso porque `lint-staged@17.0.7` requer Node `>=22.22.1`, enquanto o ambiente está em Node `22.22.0`; as validações usadas nesta mudança executaram normalmente.
