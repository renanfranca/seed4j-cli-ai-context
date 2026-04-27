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

## Desired End State

- `seed4j --version` e `seed4j list` em `extension mode` funcionam com extensao Spring Boot fat jar.
- O child process recebe `loader.path` com entradas compativeis com fat jar:
  - `jar:file:<extension.jar>!/BOOT-INF/classes`
  - `jar:file:<extension.jar>!/BOOT-INF/lib` (ou equivalente com sufixo `/`)
- Se o `extension.jar` nao tiver `BOOT-INF/classes`, o CLI encerra com erro de runtime configuracao invalida e mensagem objetiva.
- Extensao em jar flat deixa de ser suportada por decisao de produto.
- Testes de unidade e integracao cobrem fluxo positivo (fat jar valido) e negativo (jar flat/invalido).

## Milestones

### Milestone 1 - Contrato de carregamento fat jar no bootstrap

#### Goal

Definir e implementar a regra central de `loader.path` para extensao fat jar.

#### Changes

- [ ] Introduzir componente de dominio no bootstrap para resolver `loader.path` de extensao (exemplo: `RuntimeExtensionLoaderPathResolver`) com foco em fat jar.
- [ ] Atualizar `Seed4JCliLauncher` para usar o resolvedor ao montar `JavaChildProcessRequest`.
- [ ] Garantir que o `loader.path` final use entradas `jar:file:...!/BOOT-INF/classes` e `jar:file:...!/BOOT-INF/lib`.
- [ ] Manter `seed4j.cli.runtime.*` inalterado para nao quebrar leitura de runtime selection.

#### Validation

- [ ] Command: `./mvnw -Dtest=Seed4JCliLauncherTest,JavaProcessChildLauncherTest test`
- [ ] Expected result: testes passam e verificam que `loader.path` foi montado no formato fat jar.

#### Acceptance Criteria

- [ ] Existe validacao automatizada para formato de `loader.path` compativel com fat jar.
- [ ] Nao existe uso residual de `loader.path=<extension.jar>` puro para extension mode.

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

Garantir cobertura de comportamento real em `-jar` com runtime extension.

#### Changes

- [ ] Atualizar `ExtensionRuntimeFixture` para gerar fixture de extensao no layout fat jar (incluindo classes de extensao em `BOOT-INF/classes` e, quando necessario, libs em `BOOT-INF/lib`).
- [ ] Ajustar testes empacotados de `list` e `--version` para usar fixture fat jar.
- [ ] Adicionar cenario negativo empacotado para jar flat/invalido em extension mode (falha esperada).

#### Validation

- [ ] Command: `./mvnw -Dit.test=ExtensionRuntimeBootstrapPackagedJarIT,ExtensionRuntimeBootstrapListPackagedJarIT failsafe:integration-test failsafe:verify`
- [ ] Expected result: cenarios positivos passam com fat jar; cenario negativo falha conforme contrato.

#### Acceptance Criteria

- [ ] Em extension mode com fat jar valido, modulo extension-only aparece no `seed4j list`.
- [ ] Em extension mode com jar sem layout fat jar, bootstrap falha cedo.

### Milestone 4 - Documentacao e validacao completa

#### Goal

Fechar com contrato documentado e trilha de verificacao completa.

#### Changes

- [ ] Atualizar `documentation/Commands.md` com contrato de runtime extension para Spring Boot fat jar.
- [ ] Documentar breaking change: suporte legado de jar flat removido.
- [ ] Incluir exemplos de erro e de setup correto com `extension.jar` fat jar.

#### Validation

- [ ] Command: `./mvnw clean verify`
- [ ] Expected result: suite completa verde (unit, integration, coverage, checkstyle).
- [ ] Command: `npm run prettier:check`
- [ ] Expected result: sem divergencias de formatacao.

#### Acceptance Criteria

- [ ] Contrato final esta claro para quem gera extensao.
- [ ] Nao ha ambiguidade entre formato suportado e formato rejeitado.

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

- Decision: `seed4j-cli` deve suportar somente extensao em Spring Boot fat jar.
  Rationale: decisao explicita de produto para padronizar empacotamento e evitar ambiguidades de runtime.
  Date/Author: 2026-04-27 / User + Codex

- Decision: comportamento para layout invalido deve ser fail-fast.
  Rationale: evita sucesso parcial silencioso e acelera diagnostico de configuracao incorreta.
  Date/Author: 2026-04-27 / User + Codex

- Decision: nao introduzir nova chave de configuracao para escolher estrategia de carregamento.
  Rationale: manter configuracao simples e reduzir superficie de erro operacional.
  Date/Author: 2026-04-27 / Codex

## Risks and Mitigations

- Risk: Quebra de compatibilidade para usuarios com extensoes legadas em jar flat.
  Mitigation: documentar breaking change com mensagem de erro clara e orientacao objetiva de migracao para fat jar.

- Risk: Montagem incorreta de URI em `loader.path` pode variar por plataforma.
  Mitigation: montar URI via API de `Path`/`URI`, cobrir testes com assercao exata do formato final e validar em testes empacotados.

- Risk: Fixture de teste fat jar ficar diferente do comportamento real de extensao produzida.
  Mitigation: gerar fixture com layout Spring Boot realista (`BOOT-INF/classes` e estrutura de manifest) e validar pelo fluxo `-jar`.

- Risk: Erro de validacao de layout reduzir observabilidade (mensagem vaga).
  Mitigation: padronizar texto de erro com arquivo alvo e expectativa explicita (`BOOT-INF/classes` ausente).

## Validation Strategy

1. Rodar testes unitarios de bootstrap e montagem de comando para validar `loader.path`.
2. Rodar testes unitarios de validacao de runtime para cenarios validos e invalidos.
3. Rodar testes empacotados de extension mode (`--version` e `list`).
4. Rodar `./mvnw clean verify` para regressao completa.
5. Rodar `npm run prettier:check` para garantir formatacao consistente.
6. Validacao manual final:
   - configurar `~/.config/seed4j-cli.yml` com `mode: extension`,
   - instalar fat jar em `~/.config/seed4j-cli/runtime/active/extension.jar`,
   - executar `seed4j list` e confirmar presenca de modulo de extensao.

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

- Preencher durante a execucao com descobertas nao obvias sobre `PropertiesLauncher`, URIs `jar:file:` e comportamento de classpath em runtime extension.
