# Corrigir `seed4j --version` em `extension mode` (sem `null` e sem warnings de Logback)

Este ExecPlan e um documento vivo. Atualize `Progress`, `Decisions`, `Risks` e `Lessons Learned` durante a execucao.

## Purpose / Big Picture

Hoje, quando o usuario roda `seed4j --version` com `seed4j.runtime.mode: extension`, a saida exibe warnings de Logback e mostra `Seed4J CLI vnull` / `Seed4J version: null`. Isso quebra a confiabilidade do comando mais basico de diagnostico.
Este plano corrige o bootstrap de runtime para que o comando sempre mostre versoes validas do CLI e permaneca silencioso (sem ruido de logging), mesmo quando a extensao publica `config/application.yml` e `logback-spring.xml` proprios.

## Scope

Escopo:

- Corrigir a origem de `project.version` e `project.seed4j-version` no child process em `extension mode`, removendo dependencia de propriedades que podem ser sobrescritas pela extensao.
- Corrigir selecao de configuracao de logging do child process para evitar que `logback-spring.xml` da extensao sobrescreva o baseline do CLI.
- Adicionar testes (unitarios e/ou de integracao empacotada) que reproduzem e blindam os dois sintomas.
- Validar manualmente com `seed4j --version` em `extension mode`.

Fora de escopo:

- Redesenhar o modelo de extensoes (`loader.path`, metadata ou compatibilidade de modulos).
- Alterar UX de outros comandos alem do impacto indireto do bootstrap.
- Trocar stack de logging.

## Definitions

- `extension mode`: modo ativado por `~/.config/seed4j-cli.yml` com `seed4j.runtime.mode: extension`.
- `child process`: JVM filha iniciada por `Seed4JCliLauncher` via `PropertiesLauncher`.
- `resource shadowing`: quando recurso de classpath da extensao (por exemplo `config/application.yml`) e carregado antes do mesmo recurso do CLI.
- `baseline de logging`: configuracao de log minima esperada para comandos CLI (sem ruido de bootstrap por padrao).

## Existing Context

- O comando `--version` formata versoes usando `@Value("${project.version}")` e `@Value("${project.seed4j-version}")` em `src/main/java/com/seed4j/cli/command/infrastructure/primary/Seed4JCommandsFactory.java`.
- Essas propriedades sao definidas em `src/main/resources/config/application.yml` via filtragem Maven:
  - `project.version: '@project.version@'`
  - `project.seed4j-version: '@seed4j.version@'`
- Em `extension mode`, o child process injeta `loader.path` para `BOOT-INF/classes` e `BOOT-INF/lib` da extensao e injeta `logging.config=classpath:logback-spring.xml` em `src/main/java/com/seed4j/cli/bootstrap/domain/Seed4JCliLauncher.java`.
- Com esse classpath, o `config/application.yml` e o `logback-spring.xml` da extensao podem ser resolvidos antes dos do CLI.
- Evidencia observada no ambiente local (2026-04-28):
  - `seed4j --version` imprime:
    - `Missing watchable .xml or .properties files`
    - `Watching .xml files requires that the main configuration file is reachable as a URL`
    - `Seed4J CLI vnull`
    - `Seed4J version: null`
  - Em `standard mode`, o mesmo comando imprime versoes corretas.

## Desired End State

- Em `extension mode`, `seed4j --version` imprime:
  - `Seed4J CLI v<versao-valida>`
  - `Seed4J version: <versao-valida>`
  - sem warnings de Logback no stderr/stdout.
- A origem das versoes do CLI fica robusta contra `resource shadowing` de `config/application.yml`.
- O baseline de logging do child process em `extension mode` passa a usar configuracao do proprio CLI de forma deterministica.
- Testes automatizados cobrem regressao dos dois sintomas.

## Milestones

### Milestone 1 - Reproducao automatizada da regressao atual

#### Goal

Transformar o bug observado manualmente em cenario de teste reproduzivel.

#### Changes

- [x] Adicionar/ajustar teste em `src/test/java/com/seed4j/cli/bootstrap/domain` que simula `extension mode` com extensao contendo:
  - `config/application.yml` sem chaves `project.*`.
  - `logback-spring.xml` com `scan="true"`.
- [x] Garantir que o teste capture saida de `--version` e valide os sintomas atuais (antes da correcao) para evitar falso positivo.

#### Validation

- [x] Command: `./mvnw -Dtest=ExtensionRuntimeBootstrapPackagedJarIT,ExtensionRuntimeBootstrapListPackagedJarIT failsafe:integration-test failsafe:verify`
- [x] Expected result: cenario de regressao atual e observavel e falha antes da correcao.

#### Acceptance Criteria

- [x] Existe ao menos um teste vermelho que reproduz `vnull` ou warning de logback sob `extension mode`.
- [x] O cenario usa fluxo real de child process (nao apenas teste in-process).

### Milestone 2 - Blindar origem de versao do CLI contra shadowing da extensao

#### Goal

Garantir que `Seed4JCommandsFactory` nao dependa de `project.*` carregado do classpath da extensao.

#### Changes

- [x] Introduzir fonte explicita de versao do CLI para `extension mode` (ex.: system properties publicadas pelo parent launcher).
- [x] Em `Seed4JCliLauncher`, publicar propriedades dedicadas com versao do CLI e versao do seed4j antes de iniciar child process.
- [x] Em `Seed4JCommandsFactory`, substituir `@Value("${project.*}")` direto por leitura robusta com fallback definido.
- [x] Definir fallback seguro e nao-nulo para qualquer cenario de metadado ausente.

#### Validation

- [x] Command: `./mvnw -Dtest=Seed4JCommandsFactoryTest,Seed4JCliLauncherTest test`
- [x] Expected result: versoes nunca ficam `null` no output formatado.

#### Acceptance Criteria

- [x] `Seed4J CLI vnull` nao ocorre mais.
- [x] `Seed4J version: null` nao ocorre mais.
- [x] Testes cobrem o caso onde `config/application.yml` da extensao nao contem `project.*`.

### Milestone 3 - Tornar o `logging.config` deterministico para o CLI

#### Goal

Impedir que `logback-spring.xml` da extensao seja selecionado quando o child process roda comandos do CLI.

#### Changes

- [x] Ajustar `Seed4JCliLauncher` para apontar `logging.config` para recurso exclusivo do CLI (nao ambiguo no classpath).
- [x] Caso necessario, criar arquivo de logback dedicado do CLI com nome unico em `src/main/resources`.
- [x] Manter nivel de log operacional atual (`logging.level.root=ERROR`) e sem startup info em `extension mode`.

#### Validation

- [x] Command: `./mvnw -Dtest=Seed4JCliLauncherTest,ExtensionRuntimeBootstrapInProcessTest test`
- [x] Expected result: launcher publica `logging.config` dedicado do CLI e baseline de logging permanece estavel em `extension mode`.
- [x] Command: `./mvnw -Dtest=ExtensionRuntimeBootstrapPackagedJarIT test`
- [x] Expected result: cenario empacotado com `logback-spring.xml` da extensao (`scan="true"`) nao emite warnings de watchable/URL na saida de `--version`.

#### Acceptance Criteria

- [x] Warnings de Logback deixam de aparecer no cenario de extensao com `scan="true"` no logback da propria extensao.
- [x] Configuracao de logging do CLI segue estavel e previsivel.

### Milestone 4 - Fechamento com validacao end-to-end e documentacao curta

#### Goal

Fechar a correcao com validacao local completa e registro objetivo do comportamento final.

#### Changes

- [ ] Executar suite alvo e verificacao completa do repositorio.
- [ ] Atualizar documentacao tecnica minima (ou nota no proprio ExecPlan) com comando real de verificacao manual.

#### Validation

- [ ] Command: `./mvnw clean verify`
- [ ] Expected result: build verde com cobertura e integracao passando.
- [ ] Command: `seed4j --version`
- [ ] Expected result: em `extension mode`, sem warnings de logback e sem `null` nas versoes.

#### Acceptance Criteria

- [ ] Correcao comprovada por teste automatizado e verificacao manual.
- [ ] Plano atualizado com status final e decisoes tomadas.

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

- Decision: Tratar `project.*` como metadado de build do CLI, nao como configuracao mutavel da extensao.
  Rationale: evita `null` quando `application.yml` da extensao sombreia o do CLI.
  Date/Author: 2026-04-28 / Codex

- Decision: Tornar `logging.config` de `extension mode` nao ambiguo no classpath.
  Rationale: evita selecao acidental de `logback-spring.xml` da extensao.
  Date/Author: 2026-04-28 / Codex

- Decision: Publicar `seed4j.cli.version` e `seed4j.cli.seed4j.version` no launcher antes do child process.
  Rationale: remove dependencia de `project.*` no classpath da extensao para resolver versoes.
  Date/Author: 2026-04-28 / Codex

- Decision: Em caso de metadado ausente, usar fallback textual `unknown` no output de versao.
  Rationale: evita regressao para `null` e mantem diagnostico operacional legivel.
  Date/Author: 2026-04-28 / Codex

- Decision: usar `logging.config=classpath:seed4j-cli-logback-spring.xml` no child process de `extension mode`.
  Rationale: remove ambiguidade com `logback-spring.xml` da extensao sem perder extensoes de configuracao do Spring Boot no logback do CLI.
  Date/Author: 2026-04-28 / Codex

## Risks and Mitigations

- Risk: Acoplamento excessivo entre parent e child process ao introduzir novas system properties.
  Mitigation: encapsular propriedades em tipo dedicado e cobrir com testes de contrato no launcher.

- Risk: Ajuste de `logging.config` quebrar fluxo in-process de testes.
  Mitigation: validar cenarios empacotados e in-process antes do `clean verify`.

- Risk: Extensao futura publicar recurso com mesmo nome do arquivo dedicado de logback.
  Mitigation: usar nome altamente especifico ao CLI ou localizacao explicita via URL do jar do executavel.

- Risk: Regressao silenciosa em distribuicoes instaladas (ex.: `/usr/local/bin/seed4j.jar`) sem atualizar artefato.
  Mitigation: adicionar passo explicito de empacotar/instalar e validar o binario instalado apos merge.

## Validation Strategy

1. Rodar testes focados de bootstrap e version output.
2. Rodar testes de integracao empacotada para `extension mode`.
3. Rodar `./mvnw clean verify`.
4. Validar manualmente `seed4j --version` com `~/.config/seed4j-cli.yml` em `extension`.

## Rollout and Recovery

- Rollout:
  - Gerar novo `seed4j.jar` com a correcao.
  - Atualizar instalacao local/global que fornece o comando `seed4j`.
  - Validar `seed4j --version` em `standard` e `extension`.
- Recovery:
  - Reverter commit da correcao se regressao aparecer.
  - Restaurar binario anterior instalado e revalidar comando.

## Lessons Learned

- Em runtime com `PropertiesLauncher` + `loader.path`, recursos de configuracao da extensao podem ter precedencia sobre recursos do CLI.
- Confiar em `@Value` de chaves definidas em `application.yml` generico e fragil quando o classpath inclui extensoes.
- Os ITs empacotados dependem de um artefato `target/seed4j-cli-*.jar` existente; para reproducao local consistente, rodar `./mvnw -DskipTests package` antes da suite de failsafe.
- Publicar versao em system property no parent launcher e mais robusto do que ler `project.*` diretamente no child quando ha shadowing de resources.
- Fallback explicito para `unknown` impede saida `null` mesmo quando metadados de build nao estao disponiveis.
- Rodar `package` e testes empacotados em paralelo pode causar falso negativo por corrida no artefato em `target/`; checkpoint vertical deve ser sequencial.
