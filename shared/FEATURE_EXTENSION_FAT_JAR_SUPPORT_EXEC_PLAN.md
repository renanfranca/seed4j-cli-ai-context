# Suportar extensao Spring Boot fat jar no runtime `extension` do seed4j-cli

Este ExecPlan e um documento vivo. Atualize `Progress`, `Decisions`, `Risks` e `Lessons Learned` durante a execucao.

## Purpose / Big Picture

Hoje o `seed4j-cli` entra em `extension mode`, mas nao consegue descobrir modulos quando o `extension.jar` foi gerado como Spring Boot fat jar (conteudo em `BOOT-INF/classes` e `BOOT-INF/lib`). Isso faz o usuario configurar tudo corretamente e ainda assim nao ver seus modulos no `seed4j list`.
Este plano entrega suporte explicito a esse formato de extensao, com validacao observavel em runtime: com fat jar valido os modulos aparecem; com artefato invalido o CLI falha cedo com mensagem clara.

## Scope

Escopo:

- Ajustar bootstrap de child process para montar `loader.path` compativel com Spring Boot fat jar.
- Validar layout do `extension.jar` antes de montar o child process e falhar rapido em caso de inconsistencias.
- Atualizar fixtures e testes para validar o caminho real (`-jar` + `PropertiesLauncher`) com extensao em fat jar.
- Blindar o logging do child process para evitar ruido quando a extensao inclui `config/application.yml` e `logback-spring.xml`.
- Atualizar documentacao operacional do runtime extension.

Fora de escopo:

- Manter compatibilidade com extensao legada em jar flat (sem `BOOT-INF/*`).
- Alterar contrato de localizacao de `metadata.yml` e `extension.jar`.
- Alterar UX dos comandos (`list`, `apply`) alem do comportamento derivado do carregamento de modulos.

## Definitions

- `extension mode`: modo de runtime ativado por `seed4j.runtime.mode: extension` em `~/.config/seed4j-cli.yml`.
- `fat jar` (Spring Boot): jar com classes em `BOOT-INF/classes` e dependencias em `BOOT-INF/lib`.
- `jar flat`: jar com classes em raiz (`com/...`) sem layout `BOOT-INF/*`.
- `loader.path`: system property lida por `org.springframework.boot.loader.launch.PropertiesLauncher` para incluir classpath adicional.
- `fail-fast`: interromper antes de subir Spring quando configuracao de runtime esta invalida.

## Existing Context

- O launcher monta child process com `PropertiesLauncher` e publica `loader.path` em:
  - `src/main/java/com/seed4j/cli/bootstrap/domain/Seed4JCliLauncher.java`
  - `src/main/java/com/seed4j/cli/bootstrap/domain/JavaProcessChildLauncher.java`
- O path atual de extensao no `loader.path` aponta so para o jar raiz (`.../extension.jar`), o que nao descobre classes em `BOOT-INF/classes` de fat jar.
- A validacao atual de runtime confere existencia de `metadata.yml` e `extension.jar`, mas nao valida layout interno do jar:
  - `src/main/java/com/seed4j/cli/bootstrap/domain/RuntimeSelection.java`
- Fixtures e testes atuais assumem jar minimo ou jar flat para extensao:
  - `src/test/java/com/seed4j/cli/bootstrap/domain/ExtensionRuntimeFixture.java`
  - `src/test/java/com/seed4j/cli/bootstrap/domain/ExtensionRuntimeBootstrapListPackagedJarIT.java`
  - `src/test/java/com/seed4j/cli/bootstrap/domain/Seed4JCliLauncherTest.java`
- O guia de comandos documenta extension mode e metadata, mas ainda nao explicita contrato de fat jar:
  - `documentation/Commands.md`
- Algumas extensoes empacotam `config/application.yml` e `logback-spring.xml`; com `loader.path` em `BOOT-INF/classes`, esses arquivos podem sobrescrever niveis de log e nome da aplicacao do CLI.

## Desired End State

- `seed4j --version` e `seed4j list` em `extension mode` funcionam com extensao Spring Boot fat jar.
- O child process recebe `loader.path` com entradas compativeis com fat jar:
  - `jar:file:<extension.jar>!/BOOT-INF/classes`
  - `jar:file:<extension.jar>!/BOOT-INF/lib` (ou equivalente com sufixo `/`)
- Se o `extension.jar` nao tiver `BOOT-INF/classes`, o CLI encerra com erro de runtime configuracao invalida e mensagem objetiva.
- Extensao em jar flat deixa de ser suportada por decisao de produto.
- Em `extension mode`, `seed4j list` e `seed4j --version` nao exibem logs de startup/warnings de logback por override de logging da extensao.
- Testes de unidade e integracao cobrem fluxo positivo (fat jar valido) e negativo (jar flat/invalido).

## Milestones

### Milestone 1 - Contrato de carregamento fat jar no bootstrap

#### Goal

Definir e implementar a regra central de `loader.path` para extensao fat jar.

#### Changes

- [x] Introduzir componente de dominio no bootstrap para resolver `loader.path` de extensao (exemplo: `RuntimeExtensionLoaderPathResolver`) com foco em fat jar.
- [x] Atualizar `Seed4JCliLauncher` para usar o resolvedor ao montar `JavaChildProcessRequest`.
- [x] Garantir que o `loader.path` final use entradas `jar:file:...!/BOOT-INF/classes` e `jar:file:...!/BOOT-INF/lib`.
- [x] Manter `seed4j.cli.runtime.*` inalterado para nao quebrar leitura de runtime selection.

#### Validation

- [x] Command: `./mvnw -Dtest=Seed4JCliLauncherTest,JavaProcessChildLauncherTest test`
- [x] Expected result: testes passam e verificam que `loader.path` foi montado no formato fat jar.

#### Acceptance Criteria

- [x] Existe validacao automatizada para formato de `loader.path` compativel com fat jar.
- [x] Nao existe uso residual de `loader.path=<extension.jar>` puro para extension mode.

### Milestone 2 - Fail-fast para layout de extensao invalido

#### Goal

Evitar comportamento silencioso quando o jar nao atende contrato de fat jar.

#### Changes

- [ ] Adicionar validacao explicita do layout interno do `extension.jar` durante resolucao de runtime (checar presenca de `BOOT-INF/classes`).
- [ ] Emitir `InvalidRuntimeConfigurationException` com mensagem orientada a usuario quando layout nao for fat jar valido.
- [ ] Garantir que o erro aconteca antes de tentar subir child process.

#### Validation

- [ ] Command: `./mvnw -Dtest=RuntimeSelectionTest,Seed4JCliLauncherTest test`
- [ ] Expected result: cenario com jar invalido falha com erro de runtime configuracao; cenario valido continua verde.

#### Acceptance Criteria

- [ ] Cenario invalido retorna exit code nao-zero.
- [ ] Mensagem de erro deixa claro que o runtime espera Spring Boot fat jar (`BOOT-INF/classes`).

### Milestone 3 - Atualizar fixtures e testes empacotados para fat jar

#### Goal

Garantir cobertura de comportamento real em `-jar` com runtime extension e fechar regressao aberta pelo fail-fast do milestone 2 nos testes de fixture/runtime in-process.

#### Changes

- [x] Atualizar `ExtensionRuntimeFixture` para gerar fixture de extensao no layout fat jar (incluindo classes de extensao em `BOOT-INF/classes` e, quando necessario, libs em `BOOT-INF/lib`).
- [x] Ajustar testes empacotados de `list` e `--version` para usar fixture fat jar.
- [x] Adicionar cenario negativo empacotado para jar flat/invalido em extension mode (falha esperada).
- [x] Ajustar `ExtensionRuntimeFixtureTest` e `ExtensionRuntimeBootstrapInProcessTest` para usar fixture fat jar e voltar o `clean verify` para verde apos a validacao de layout.

#### Validation

- [x] Command: `./mvnw -Dtest=ExtensionRuntimeFixtureTest,ExtensionRuntimeBootstrapInProcessTest test`
- [x] Expected result: testes de fixture e runtime in-process passam com fat jar valido e falham de forma controlada no cenario invalido.
- [x] Command: `./mvnw -Dit.test=ExtensionRuntimeBootstrapPackagedJarIT,ExtensionRuntimeBootstrapListPackagedJarIT failsafe:integration-test failsafe:verify`
- [x] Expected result: cenarios positivos passam com fat jar; cenario negativo falha conforme contrato.

#### Acceptance Criteria

- [x] Em extension mode com fat jar valido, modulo extension-only aparece no `seed4j list`.
- [x] Em extension mode com jar sem layout fat jar, bootstrap falha cedo.

### Milestone 4 - Blindagem de logging do CLI contra overrides da extensao

#### Goal

Garantir saida operacional limpa do CLI em `extension mode`, mesmo quando a extensao carrega configuracao propria de logging.

#### Changes

- [x] Definir contrato de logging do child process em `extension mode` (sem logs de startup/info por padrao).
- [x] Atualizar bootstrap para aplicar baseline de logging com precedencia sobre `logging.*` e `logback-spring.xml` da extensao.
- [x] Garantir que overrides da extensao nao alterem a observabilidade padrao do CLI em `seed4j list` e `seed4j --version`.
- [x] Cobrir com testes o cenario onde `extension.jar` contem `config/application.yml` e `logback-spring.xml`.

#### Validation

- [x] Command: `./mvnw -Dtest=ExtensionRuntimeBootstrapInProcessTest,Seed4JCliLauncherTest test`
- [x] Expected result: comandos em `extension mode` nao exibem logs de startup/info nem warnings de configuracao de logback.

#### Acceptance Criteria

- [x] Em `extension mode`, a saida dos comandos permanece focada em resultado funcional (sem ruido de bootstrap/logback por padrao).
- [x] O comportamento e resiliente mesmo quando a extensao publica overrides de `logging.*`.

### Milestone 5 - Cobertura de validacao do layout do extension jar

#### Goal

Eliminar o gap de cobertura no `RuntimeExtensionJarLayoutValidator` sem remover ramos uteis de protecao.

#### Changes

- [x] Cobrir o caminho de `IOException` na leitura do `extension.jar` para validar fallback com `InvalidRuntimeConfigurationException`.
- [x] Cobrir o ramo de entrada `BOOT-INF/classes` sem `/` final.
- [x] Cobrir o ramo onde o jar contem apenas entradas filhas (`BOOT-INF/classes/...`) sem entrada explicita de diretorio.
- [x] Garantir que `./mvnw clean verify` deixa de falhar por cobertura nessa classe.

#### Validation

- [x] Command: `./mvnw -Dtest=RuntimeSelectionTest test`
- [x] Expected result: cenarios validos/invalidos de layout continuam corretos e os novos ramos ficam exercitados.
- [x] Command: `./mvnw clean verify`
- [x] Expected result: sem violacao de cobertura para `RuntimeExtensionJarLayoutValidator`.

#### Acceptance Criteria

- [x] A classe `RuntimeExtensionJarLayoutValidator` fica com branch/line coverage em 100%.
- [x] Nao houve remocao de ramos defensivos apenas para satisfazer cobertura.

### Milestone 6 - Documentacao e validacao completa

#### Goal

Fechar com contrato documentado e trilha de verificacao completa.

#### Changes

- [x] Atualizar `documentation/Commands.md` com contrato de runtime extension para Spring Boot fat jar.
- [x] Documentar breaking change: suporte legado de jar flat removido.
- [x] Incluir exemplos de erro e de setup correto com `extension.jar` fat jar.

#### Validation

- [x] Command: `./mvnw clean verify`
- [x] Expected result: suite completa verde (unit, integration, coverage, checkstyle).
- [x] Command: `npm run prettier:check`
- [x] Expected result: sem divergencias de formatacao.

#### Acceptance Criteria

- [x] Contrato final esta claro para quem gera extensao.
- [x] Nao ha ambiguidade entre formato suportado e formato rejeitado.

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
- [x] Milestone 6 started
- [x] Milestone 6 completed

## Decisions

- Decision: `seed4j-cli` deve suportar somente extensao em Spring Boot fat jar.
  Rationale: decisao explicita de produto para padronizar empacotamento e evitar ambiguidades de runtime.
  Date/Author: 2026-04-27 / User + Codex

- Decision: comportamento para layout invalido deve ser fail-fast.
  Rationale: evita sucesso parcial silencioso e acelera diagnostico de configuracao incorreta.
  Date/Author: 2026-04-27 / User + Codex

- Decision: nao introduzir nova chave de configuracao para escolher estrategia de carregamento.
  Rationale: manter configuracao simples e reduzir superficie de erro operacional.
  Date/Author: 2026-04-27 / Codex

- Decision: o CLI deve impor baseline de logging no child process em `extension mode`, independentemente dos overrides da extensao.
  Rationale: preservar UX dos comandos e evitar ruido operacional apos habilitar classpath fat jar.
  Date/Author: 2026-04-27 / User + Codex

- Decision: erros de `InvalidRuntimeConfigurationException` no bootstrap devem ser propagados ao usuario via `stderr` antes do exit code 1.
  Rationale: sem mensagem explicita, o fail-fast perde utilidade operacional e dificulta diagnostico.
  Date/Author: 2026-04-27 / User + Codex

## Risks and Mitigations

- Risk: Quebra de compatibilidade para usuarios com extensoes legadas em jar flat.
  Mitigation: documentar breaking change com mensagem de erro clara e orientacao objetiva de migracao para fat jar.

- Risk: Montagem incorreta de URI em `loader.path` pode variar por plataforma.
  Mitigation: montar URI via API de `Path`/`URI`, cobrir testes com assercao exata do formato final e validar em testes empacotados.

- Risk: Fixture de teste fat jar ficar diferente do comportamento real de extensao produzida.
  Mitigation: gerar fixture com layout Spring Boot realista (`BOOT-INF/classes` e estrutura de manifest) e validar pelo fluxo `-jar`.

- Risk: Erro de validacao de layout reduzir observabilidade (mensagem vaga).
  Mitigation: padronizar texto de erro com arquivo alvo e expectativa explicita (`BOOT-INF/classes` ausente).

- Risk: Blindagem de logging esconder sinais uteis para diagnostico.
  Mitigation: limitar supressao ao bootstrap padrao e manter logs de erro/falha de runtime.

- Risk: `logging.config` apontar para localizacao invalida em ambiente de teste/in-process e quebrar bootstrap.
  Mitigation: usar recurso estavel de classpath (`classpath:logback-spring.xml`) em vez de URL `jar:file:...` do executavel.

## Validation Strategy

1. Rodar testes unitarios de bootstrap e montagem de comando para validar `loader.path`.
2. Rodar testes unitarios de validacao de runtime para cenarios validos e invalidos.
3. Rodar testes empacotados de extension mode (`--version` e `list`).
4. Rodar testes de blindagem de logging no caminho publico de extension mode.
5. Rodar `./mvnw clean verify` para regressao completa.
6. Rodar `npm run prettier:check` para garantir formatacao consistente.
7. Validacao manual final:
   - configurar `~/.config/seed4j-cli.yml` com `mode: extension`,
   - instalar fat jar em `~/.config/seed4j-cli/runtime/active/extension.jar`,
   - executar `seed4j list` e confirmar presenca de modulo de extensao sem logs de startup indevidos.

## Rollout and Recovery

Rollout:

1. Entregar em PR unico com descricao de breaking change.
2. Anexar evidencias dos comandos de validacao local.
3. Atualizar documentacao antes do merge para evitar uso de formato legado.

Recovery:

1. Se regressao em producao impedir carregamento de fat jar valido, reverter commit de bootstrap rapidamente.
2. Se erro indicar falso positivo de layout invalido, corrigir validacao e repetir suite de bootstrap + IT empacotada.
3. Se necessario, abrir hotfix com mensagem temporaria de contingencia no runtime.

## Lessons Learned

- A baseline de logging em `extension mode` precisa ser configurada por system properties no launcher, nao por suposicao de ordem de classpath da extensao.
- Para manter paridade entre caminho empacotado e testes in-process, `logging.config` deve referenciar recurso de classpath do CLI em vez de caminho interno do jar executavel.

- Os testes empacotados (`failsafe`) dependem de um artefato CLI jar existente em `target/`; para evitar falso-negativo de ambiente, executar `./mvnw -DskipTests package` antes da rodada de IT quando o jar nao estiver presente.

- Preencher durante a execucao com descobertas nao obvias sobre `PropertiesLauncher`, URIs `jar:file:` e comportamento de classpath em runtime extension.
- Validar layout fat jar por existencia de entradas `BOOT-INF/classes` no proprio `extension.jar` evita falso positivo de artefato presente mas inutil em runtime.
- O fail-fast de runtime precisa de mensagem em `stderr`; apenas retornar exit code nao-zero nao entrega diagnostico suficiente ao usuario.
- A validacao de layout no milestone 2 quebra fixtures legados que ainda geram jar flat (`ExtensionRuntimeFixtureTest`); ajuste de fixtures/testes empacotados permanece responsabilidade do milestone 3.
