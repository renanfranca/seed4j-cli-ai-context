# Extension Mode com `loader.path` filtrado (sem mudancas no gerador `seed4j-extension`)

Este ExecPlan e um documento vivo. Atualize `Progress`, `Decisions`, `Risks` e `Lessons Learned` durante a execucao.

## Purpose / Big Picture

Hoje o `extension mode` mistura runtime completo da extensao com runtime do CLI via `loader.path`, o que permite interferencia global (por exemplo `config/application.yml` e `logback-spring.xml`) e quebra a regra aditiva de catalogo. O objetivo deste plano e manter a estrategia `loader.path` (sem worker separado) e endurecer o bootstrap para importar apenas as contribuicoes necessarias de modulo/extensao, preservando o runtime do CLI como fonte de verdade. O resultado observavel esperado e: `seed4j list` continua aditivo (nada do core some), modulos da extensao aparecem, e `seed4j apply` continua funcional para qualquer modulo visivel no catalogo final (core + extensao), com override global de readers/resources da extensao quando presentes no contexto Spring.

## Scope

Escopo:

- Manter arquitetura atual de child process com `PropertiesLauncher`.
- Implementar overlay filtrado de `BOOT-INF/classes` da extensao antes de setar `loader.path`.
- Bloquear recursos globais da extensao que afetam runtime do CLI (`application*`, `logback*`).
- Suportar extensoes com pacote customizado (`com.mycompany...`) sem alteracao no gerador.
- Definir politica robusta para `BOOT-INF/lib` da extensao sem sobrescrever infra base do CLI.
- Cobrir com testes unitarios/integracao e validacao manual.

Fora de escopo:

- Criar worker JVM separado.
- Alterar contrato de geracao no repo `seed4j` (`seed4j-extension`).
- Introduzir fallback legado para estrategia antiga de `loader.path` bruto.

Restricao explicita:

- Nao realizar mudancas no gerador `seed4j-extension` (repo `seed4j`) em nenhuma hipotese.

## Definitions

- `overlay filtrado`: diretorio local em cache contendo classes/resources extraidos de `BOOT-INF/classes`, com remocao de recursos globais de runtime.
- `recursos globais`: arquivos que alteram bootstrap/configuracao do processo inteiro (`config/application*.yml`, `config/application*.yaml`, `config/application*.properties`, `logback*.xml`).
- `catalogo aditivo`: em `extension mode`, modulos do core continuam visiveis e a extensao apenas adiciona novos slugs.
- `libs ausentes`: jars presentes em `BOOT-INF/lib` da extensao que nao existem no classpath base do CLI (comparando por coordenada inferida do nome do jar).
- `override global de readers/resources`: em `extension mode`, readers/resources de dependencias carregados pela extensao participam do mesmo contexto Spring do CLI e podem sobrescrever valores usados por qualquer `apply` (modulos do core e da extensao), conforme precedencia de bean/merge.

## Existing Context

- O launcher atual injeta `loader.path` com `jar:<ext>!/BOOT-INF/classes` e `jar:<ext>!/BOOT-INF/lib/` no processo principal:
  - `src/main/java/com/seed4j/cli/bootstrap/domain/Seed4JCliLauncher.java`
  - `src/main/java/com/seed4j/cli/bootstrap/domain/RuntimeExtensionLoaderPathResolver.java`
- O CLI sobe Spring com scan base do core+CLI, e atualmente depende de composicao de classpath para enxergar contribuicoes da extensao:
  - `src/main/java/com/seed4j/cli/Seed4JCliApp.java`
- As extensoes exemplo trazem recursos globais em `src/main/resources/config/application.yml` e `src/main/resources/logback-spring.xml`, e tambem recursos funcionais em `/generator/**` usados por readers de dependencias.
- As extensoes geradas registram readers de dependencias com `@Repository` + `@Order(Ordered.HIGHEST_PRECEDENCE)`, e o core agrega `Collection<...Reader>` com merge de versoes; por isso o override da extensao impacta globalmente o `apply` quando ha mesma chave/source.
- Investigacoes e specs existentes em `_temporary/ai_agent/seed4j-cli-ai-context/shared/` confirmam:
  - risco estrutural de classpath compartilhado;
  - necessidade de manter catalogo aditivo;
  - necessidade de suportar pacote customizado.

## Desired End State

- Em `extension mode`, o `loader.path` do child process nao aponta mais para `BOOT-INF/classes` bruto do jar da extensao.
- O bootstrap cria (ou reutiliza) cache local por hash do `extension.jar` com overlay filtrado.
- O overlay preserva classes e recursos funcionais da extensao (ex.: `generator/**`) e remove recursos globais de runtime.
- `seed4j list` em extension mode:
  - mostra todos os slugs do core;
  - adiciona slugs da extensao;
  - reflete a uniao implicita de modulos carregados no runtime Spring (core + extensao), sem merge manual de catalogo;
  - nao remove slugs do core por `hidden-resources` vindo da extensao.
- `seed4j apply` continua funcionando para modulos do core e da extensao.
- Em `extension mode`, `apply` usa o conjunto global de readers/resources ativos no contexto Spring (core + extensao).
- Overrides de readers/resources da extensao sao comportamento esperado: quando houver sobreposicao de chave/source, eles podem afetar `apply` de modulos do core e da extensao.
- Extensao com `Start-Class` em pacote nao `com.seed4j` funciona sem alteracoes no gerador.
- `BOOT-INF/lib` da extensao nao sobrescreve libs base do CLI; quando necessario, apenas libs ausentes podem ser adicionadas de forma controlada.

## Milestones

### Milestone 1 - Infra de overlay filtrado em cache

#### Goal

Criar infraestrutura de bootstrap para extrair e reutilizar classes/resources da extensao em um diretorio filtrado estavel por hash.

#### Changes

- [x] Adicionar tipo de dominio para identidade de cache da extensao (hash de `extension.jar` e metadados relevantes).
- [x] Adicionar componente para materializar overlay em `~/.config/seed4j-cli/runtime/cache/<hash>/classes`.
- [x] Garantir idempotencia: cache hit nao reextrai; cache miss extrai atomico.
- [x] Atualizar limpeza/erros para nao deixar cache parcial em caso de falha.

#### Validation

- [x] Command: `./mvnw -Dtest=RuntimeSelectionTest,Seed4JCliLauncherTest test`
- [x] Expected result: novos testes de bootstrap/cache passam e nao quebram selecao de runtime.
- [x] Additional command: `./mvnw -Dtest=RuntimeSelectionTest,Seed4JCliLauncherTest,RuntimeExtensionOverlayCacheTest test`
- [x] Additional result: suite combinada do milestone verde.

#### Acceptance Criteria

- [x] Overlay filtrado e criado em cache com estrutura previsivel.
- [x] Execucoes repetidas reutilizam cache sem reprocessar jar.

### Milestone 2 - Filtro de recursos globais e novo `loader.path`

#### Goal

Trocar o `loader.path` da extensao bruta por `loader.path` apontando para o overlay filtrado.

#### Changes

- [x] Implementar filtro de recursos globais no overlay:
  - [x] remover `config/application*.yml`;
  - [x] remover `config/application*.yaml`;
  - [x] remover `config/application*.properties`;
  - [x] remover `logback*.xml`.
- [x] Preservar recursos funcionais necessarios (ex.: `generator/**`, `messages/**`, templates e assets), incluindo os usados por readers compartilhados de dependencias.
- [x] Atualizar `RuntimeExtensionLoaderPathResolver` para receber caminho do overlay local em vez de URL `jar:` para `BOOT-INF/classes`.
- [x] Manter baseline de logging do CLI no child process.

#### Validation

- [x] Command: `./mvnw -Dtest=Seed4JCliLauncherTest,ExtensionRuntimeBootstrapInProcessTest test`
- [x] Expected result: `loader.path` aponta para overlay local filtrado e saida continua limpa.

#### Acceptance Criteria

- [x] `seed4j.hidden-resources` da extensao nao remove mais modulo do core em `list`.
- [x] Modulos da extensao continuam sendo descobertos.
- [x] Recursos usados por readers de dependencias da extensao permanecem acessiveis no overlay.

### Milestone 3 - Descoberta robusta para pacote customizado (`Start-Class`)

#### Goal

Garantir que contribuicoes da extensao sejam carregadas mesmo quando a extensao usa pacote base customizado (ex.: `com.mycompany...`).

#### Changes

- [x] Ler `Start-Class` do manifest do `extension.jar` durante bootstrap.
- [x] Publicar propriedade de runtime dedicada com `start-class` resolvido.
- [x] Ajustar inicializacao Spring no child path para incluir `spring.main.sources` apropriado sem duplicar `Seed4JCliApp`.
- [x] Cobrir cenarios:
  - [x] extensao `com.seed4j...`;
  - [x] extensao `com.mycompany...`.

#### Validation

- [x] Command: `./mvnw -Dit.test=ExtensionRuntimeBootstrapPackagedJarIT,ExtensionRuntimeBootstrapListPackagedJarIT failsafe:integration-test failsafe:verify`
- [x] Expected result: list/apply em extension mode funcionam para pacotes diferentes sem erro de bean duplicado, mantendo override global de readers/resources da extensao.

#### Acceptance Criteria

- [x] O fluxo nao depende de naming fixo `com.seed4j.extension`.
- [x] `Start-Class` ausente/invalido falha com mensagem explicita.
- [x] O comportamento global de override de readers/resources independe do pacote base da extensao.

### Milestone 4 - Politica de `BOOT-INF/lib` sem sobrescrever runtime do CLI

#### Goal

Definir e implementar politica segura para bibliotecas da extensao mantendo `loader.path` robusto.

#### Changes

- [x] Implementar estrategia default: nao importar `BOOT-INF/lib` da extensao quando todos jars ja existem no CLI.
- [x] Implementar estrategia controlada para `libs ausentes`:
  - [x] detectar jars da extensao nao presentes no CLI;
  - [x] adicionar somente esses jars ao `loader.path` (sem trocar versoes base).
- [x] Adicionar validacao/fail-fast para conflitos de coordenada com versao divergente quando detectado risco de override.
  - [x] falhar quando o proprio CLI tiver a mesma coordenada em versoes diferentes (sem depender da ordem de `Set`).
  - [x] falhar quando a extensao trouxer coordenada ja presente no CLI com versao diferente.
- [x] Refinar politica de conflito de versao entre CLI e extensao (direcao da versao).
  - [x] quando a extensao trouxer versao mais antiga da mesma coordenada ja presente no CLI, nao fazer fail-fast; manter a lib do CLI como vencedora e registrar diagnostico em DEBUG com as duas versoes e a decisao aplicada.
  - [x] quando a extensao trouxer versao mais nova da mesma coordenada do CLI, manter fail-fast com mensagem explicita de conflito.
  - [x] quando as versoes nao forem comparaveis com seguranca (ou metadado for insuficiente), manter comportamento conservador com fail-fast explicito.
- [x] Endurecer a identificacao de `coordenada/versao` para nomes de jar fora do padrao simples `<coordinate>-<version>.jar`.
  - [x] definir politica para jars sem versao inferivel no nome (ex.: `my-lib.jar`, `bundle-all.jar`) sem falso negativo silencioso.
  - [x] cobrir versoes nao numericas no prefixo (ex.: `my-lib-v1.2.3.jar`, `my-lib-RELEASE.jar`).
  - [x] cobrir classifier/sufixo no nome (ex.: `my-lib-1.2.3-jdk17.jar`) evitando falso positivo de conflito.
  - [x] cobrir nomes renomeados por shading/relocation/custom archive name onde coordenada nao coincide com o nome final (incluindo precedencia de `pom.properties` sobre nome de arquivo e diagnostico DEBUG quando houver divergencia).
    - [x] falhar explicitamente quando jar renomeado/shaded trouxer `pom.properties` incompleto (sem `groupId`/`artifactId`/`version`) para evitar fallback silencioso por nome.
    - [x] falhar explicitamente quando jar renomeado/shaded trouxer multiplos `pom.properties` com identidades conflitantes no mesmo nested jar.
  - [x] decidir e testar comportamento para variacoes de caixa/extensao (ex.: `.JAR`) e convencoes nao padrao.
- [x] Registrar decisao em runtime logs de diagnostico (nivel DEBUG) sobre quais libs foram efetivamente adicionadas, com emissao visivel quando o CLI for executado com `--debug`.

#### Validation

- [x] Command: `./mvnw -Dtest=RuntimeSelectionTest,Seed4JCliLauncherTest,ExtensionRuntimeBootstrapInProcessTest,RuntimeExtensionMissingLibrariesSelectorTest,RuntimeExtensionLoaderPathResolverTest test`
- [x] Expected result: casos com libs equivalentes continuam verdes; casos com libs ausentes exercitam adicionamento seletivo; entradas de `BOOT-INF/lib` que nao sao `.jar` sao ignoradas.
- [x] Additional command: `./mvnw -Dtest=RuntimeExtensionMissingLibrariesSelectorTest,RuntimeExtensionLoaderPathResolverTest test`
- [x] Additional result: ciclo RED->GREEN concluido para fail-fast quando extensao traz coordenada ja presente no CLI com versao divergente.
- [x] Vertical checkpoint: `./mvnw -Dtest=Seed4JCliLauncherTest#shouldLaunchTheExtensionChildProcessRequestWithLoaderPathAndActiveDistributionSystemProperties test`
- [x] Vertical result: caminho publico do launcher em extension mode segue verde com `loader.path` esperado.
- [x] Additional command: `./mvnw -DskipTests clean package && ./mvnw -Dit.test=ExtensionRuntimeBootstrapPackagedJarIT,ExtensionRuntimeBootstrapListPackagedJarIT failsafe:integration-test failsafe:verify`
- [x] Additional result: ITs empacotados de extension mode voltaram a verde apos endurecer identidade de coordenada via `pom.properties` + fallback por nome.
- [x] Additional result: entradas `BOOT-INF/lib/*.JAR` passam a ser tratadas como bibliotecas validas para adicao seletiva no `loader.path`.
- [x] Additional result: fallback por nome passou a reconhecer versao com prefixo `v` (ex.: `shared-lib-v2.0.0.jar`) para detectar conflito de versao contra o CLI.
- [x] Additional result: fallback por nome passou a reconhecer token de versao `RELEASE` (ex.: `shared-lib-RELEASE.jar`) para detectar conflito de versao contra o CLI.
- [x] Additional result: jar renomeado na extensao nao e mais adicionado quando `pom.properties` indica mesma coordenada+versao ja presente no CLI.
- [x] Additional result: nomes com sufixo/classifier no fallback por nome (ex.: `shared-lib-1.0.0-jdk17.jar`) nao disparam mais conflito falso com `shared-lib-1.0.0.jar` do CLI.
- [x] Additional result: jars sem identidade inferivel e com mesmo nome ja presente no CLI (ex.: `bundle-all.jar`) agora falham explicitamente para evitar falso negativo silencioso.
- [x] Additional command: `./mvnw -Dtest=RuntimeExtensionMissingLibrariesSelectorTest,RuntimeExtensionLoaderPathResolverTest test`
- [x] Additional result: quando a extensao traz `fileName` igual ao CLI mas identidade conhecida diferente (`coordenada+versao`), a lib passa a ser tratada como ausente e adicionada seletivamente (nome de arquivo usado apenas como fallback sem identidade).
- [x] Additional command: `./mvnw -Dtest=RuntimeExtensionLoaderPathResolverTest,RuntimeExtensionMissingLibrariesSelectorTest test`
- [x] Additional result: quando jar renomeado/shaded da extensao contem `pom.properties` incompleto (sem `groupId`/`artifactId`/`version`), o bootstrap passa a falhar explicitamente com diagnostico em vez de seguir fallback silencioso por nome.
- [x] Additional command: `./mvnw -Dtest=RuntimeExtensionLoaderPathResolverTest,RuntimeExtensionMissingLibrariesSelectorTest test`
- [x] Additional result: quando jar renomeado/shaded da extensao contem multiplos `pom.properties` com identidades divergentes, o bootstrap passa a falhar explicitamente com diagnostico de metadado conflitante.
- [x] Additional command: `./mvnw -Dtest=RuntimeExtensionLoaderPathResolverTest,RuntimeExtensionMissingLibrariesSelectorTest test`
- [x] Additional result: mensagem de conflito de metadado em jar renomeado/shaded passa a listar todas as identidades distintas detectadas no nested jar (nao apenas as duas primeiras), com validacao consolidada apos leitura completa.
- [x] Additional command: `./mvnw -Dtest=RuntimeExtensionMissingLibrariesSelectorTest,RuntimeExtensionLoaderPathResolverTest,CliRuntimeLibraryIndexTest test`
- [x] Additional result: `RuntimeExtensionLoaderPathResolver` segue emitindo em DEBUG as libs efetivamente adicionadas ao `loader.path` (com nomes de jars) em extension mode.
- [x] Additional result: conflito CLI x extensao com versao mais antiga na extensao (ex.: `logback-classic` 1.5.22 na extensao vs 1.5.32 no CLI) nao bloqueia bootstrap; diagnostico DEBUG da decisao `CLI vence` foi adicionado e validado.
- [x] Additional result: conflito por versoes nao comparaveis passou a falhar com mensagem explicita de diagnostico (`not safely comparable`) mantendo comportamento conservador de bloqueio.
- [x] Additional result: quando `pom.properties` diverge da identidade inferida pelo nome do arquivo (mesma lib com versao divergente entre nome e metadata), a precedencia de `pom.properties` foi validada e o resolver passou a emitir DEBUG explicando o override.
- [x] Additional command: `./mvnw -Dtest=RuntimeSelectionTest,Seed4JCliLauncherTest,RuntimeExtensionMissingLibrariesSelectorTest,RuntimeExtensionLoaderPathResolverTest,CliRuntimeLibraryIndexTest test`
- [x] Additional result: em extension mode com `--debug`, o launcher nao força `logging.level.root=ERROR` e publica `logging.level.com.seed4j.cli.bootstrap.domain=DEBUG`, permitindo emissao observavel dos diagnosticos de bootstrap sem alterar o baseline de execucao sem debug.
- [x] Vertical checkpoint: `./mvnw -Dtest=Seed4JCliLauncherTest#shouldLaunchTheExtensionChildProcessRequestWithLoaderPathAndActiveDistributionSystemProperties test`
- [x] Vertical result: caminho publico do launcher segue verde apos o ajuste de `--debug` em extension mode.
- [ ] Pending manual validation: executar `seed4j --version` com extensao real contendo versao mais antiga para confirmar ausencia de travamento e emissao de diagnostico com `--debug`.

#### Acceptance Criteria

- [ ] O classpath do CLI permanece fonte principal da infraestrutura.
- [ ] Extensao so adiciona libs quando realmente ausentes.
- [x] Conflitos de versao por coordenada seguem politica explicita: conflito interno no CLI falha; entre CLI e extensao, versao mais antiga da extensao nao bloqueia (CLI vence) e versao mais nova da extensao falha com diagnostico.
- [x] A validacao de conflitos nao depende da ordem de iteracao de `Set`.
- [x] Casos de naming nao padrao em `BOOT-INF/lib` possuem politica explicita e testes cobrindo falso positivo/negativo.

#### Milestone 4 Learnings (2026-05-11)

- `./mvnw clean verify` passou em unit tests e falhou em ITs empacotados de extension mode com `exit code = 1` apos introduzir fail-fast de conflito interno no CLI.
- Causa observada: heuristica atual baseada apenas em nome de jar (`<coordinate>-<version>.jar`) detectou conflito interno no JAR empacotado para `jackson-core` e `jackson-databind` (`2.21.2` e `3.1.2`), embora sejam artefatos de grupos distintos no classpath efetivo.
- Aprendizado: validar conflito apenas por `artifactId` inferido do nome do arquivo gera falso positivo relevante em runtime real; o milestone 4 precisa evoluir a identificacao de coordenada para evitar bloquear extension mode nesses cenarios.
- Aprendizado (2026-05-13): no cenario real de `seed4j --version` com extensao trazendo `ch.qos.logback:logback-classic` 1.5.22 e CLI em 1.5.32, o fail-fast atual bloqueia um caso potencialmente seguro de downgrade da extensao; o plano passa a separar downgrade (nao bloqueante, CLI vence) de upgrade (bloqueante).
- Aprendizado (2026-05-14): manter downgrade nao bloqueante sem observabilidade reduz confianca operacional; o milestone passou a registrar em DEBUG a decisao `CLI vence` com coordenada e versoes.
- Aprendizado (2026-05-14): conflito por versoes nao comparaveis precisava mensagem dedicada; usar texto generico de conflito dificultava diagnostico, entao o fail-fast conservador passou a explicar explicitamente que as versoes nao sao comparaveis com seguranca.
- Aprendizado (2026-05-14): para jars renomeados/shaded, apenas aplicar precedencia de `pom.properties` nao era suficiente para suporte operacional; incluir log DEBUG quando metadata diverge do nome do arquivo torna a decisao auditavel.
- Aprendizado (2026-05-15): manter `logging.level.root=ERROR` de forma incondicional em extension mode anulava o valor operativo de `--debug` para o milestone 4; o launcher agora relaxa esse override no modo debug e publica nivel DEBUG apenas para o pacote de bootstrap.

### Milestone 5 - Regressao funcional fim-a-fim + documentacao

#### Goal

Fechar a mudanca com cobertura automatizada e roteiro operacional claro.

#### Changes

- [ ] Adicionar/atualizar ITs empacotados para provar:
  - [ ] `list` aditivo (core intacto + slugs da extensao);
  - [ ] `apply` de modulo do core refletindo override de readers/resources da extensao quando houver sobreposicao;
  - [ ] `apply` de modulo da extensao funcional com o mesmo conjunto global de readers/resources;
  - [ ] `--version` sem regressao de logging/versao.
- [ ] Atualizar documentacao de extension mode no repo `seed4j-cli` para refletir overlay filtrado + politica de libs.
- [ ] Registrar exemplos de falha com mensagens esperadas.

#### Validation

- [ ] Command: `./mvnw clean verify`
- [ ] Expected result: build verde completo com cobertura/checkstyle.
- [ ] Command: `npm run prettier:check`
- [ ] Expected result: sem violacoes de formatacao.

#### Acceptance Criteria

- [ ] Mudanca validada com testes e cenario manual reproduzivel.
- [ ] Documentacao operacional alinhada ao comportamento final.
- [ ] Contrato de override global no `apply` explicitado e validado.

## Progress

- [x] Milestone 1 started
- [x] Milestone 1 completed
- [x] Milestone 2 started
- [x] Milestone 2 completed
- [x] Milestone 3 started
- [x] Milestone 3 completed
- [x] Milestone 4 started
- [ ] Milestone 4 completed
- [ ] Milestone 5 started
- [ ] Milestone 5 completed

## Decisions

- Decision: Manter `loader.path` como mecanismo principal de extensao no `extension mode`.
  Rationale: atende a diretriz do produto sem introduzir worker separado.
  Date/Author: 2026-04-29 / User + Codex

- Decision: Nao alterar o gerador `seed4j-extension` no repo `seed4j`.
  Rationale: restricao explicita de escopo e governanca.
  Date/Author: 2026-04-29 / User

- Decision: Runtime do CLI e a base de infraestrutura; extensao adiciona contribuicoes de modulo e recursos funcionais.
  Rationale: reduzir risco de conflito de versao e manter previsibilidade operacional.
  Date/Author: 2026-04-29 / User + Codex

- Decision: Recursos globais de configuracao/logging da extensao nao podem participar do bootstrap do processo CLI.
  Rationale: preservar catalogo aditivo e estabilidade de logging/configuracao.
  Date/Author: 2026-04-29 / Codex

- Decision: Override global de readers/resources da extensao no `apply` e comportamento esperado em `extension mode`.
  Rationale: o contexto Spring e compartilhado (core + extensao), e a resolucao de dependencias usa colecoes ordenadas + merge; o plano deve assumir esse contrato e testa-lo explicitamente.
  Date/Author: 2026-04-30 / User + Codex

- Decision: Identidade de cache do overlay usa `SHA-256(extension.jar)` com prefixo de versao de layout (`overlay-v1`).
  Rationale: permite reuso deterministico por conteudo e invalidacao controlada quando o layout do cache evoluir.
  Date/Author: 2026-04-30 / Codex

- Decision: Selecao de libs ausentes em `BOOT-INF/lib` compara nomes de arquivo (`*.jar`) entre extensao e CLI para compor o `loader.path` sem sobrescrever jars ja existentes.
  Rationale: entrega comportamento aditivo minimo com cobertura de branch para entradas nao-jar; fail-fast para conflito entre CLI e extensao foi adicionado via heuristica de nome (`<coordinate>-<version>.jar`), e o caso de conflito interno no proprio CLI ainda permanece pendente.
  Date/Author: 2026-05-05 / Codex

- Decision: Em conflito de coordenada entre CLI e extensao com versoes divergentes, o bootstrap deve falhar antes de montar `loader.path`.
  Rationale: evita override silencioso de runtime e torna o erro diagnostico/acaoavel durante o bootstrap.
  Date/Author: 2026-05-05 / Codex

- Decision: Identidade de coordenada para `BOOT-INF/lib` deve priorizar `META-INF/maven/**/pom.properties` do jar aninhado (groupId:artifactId:version), com fallback para heuristica por nome quando metadados nao existirem.
  Rationale: evita falso positivo relevante quando o classpath contem mesmo `artifactId` em grupos distintos (ex.: `jackson-core` 2.x e 3.x).
  Date/Author: 2026-05-11 / User + Codex

- Decision: Descoberta de bibliotecas em `BOOT-INF/lib` deve tratar extensao de arquivo `.jar` de forma case-insensitive (ex.: `.JAR`).
  Rationale: evita falso negativo de libs ausentes em empacotamentos com convencao nao padrao de caixa.
  Date/Author: 2026-05-11 / User + Codex

- Decision: Selecao de `libs ausentes` deve priorizar identidade (`coordenada+versao`) quando disponivel e usar nome de arquivo apenas como fallback.
  Rationale: evita adicionar jar renomeado/reempacotado quando a mesma biblioteca ja esta presente no CLI com a mesma identidade.
  Date/Author: 2026-05-11 / User + Codex

- Decision: Fallback de identidade por nome de jar deve ser conservador e nao inferir versao para nomes com sufixo/classifier apos token numerico (ex.: `1.0.0-jdk17`).
  Rationale: reduz falso positivo de conflito quando o nome do arquivo carrega classifier/sufixo e nao representa troca real de versao da mesma coordenada.
  Date/Author: 2026-05-11 / User + Codex

- Decision: Quando a extensao trouxer jar sem identidade inferivel e com mesmo nome de arquivo ja presente no CLI, o bootstrap deve falhar explicitamente.
  Rationale: evita falso negativo silencioso em que a lib da extensao seria ignorada apenas por colisao de nome, sem garantia de equivalencia real.
  Date/Author: 2026-05-11 / User + Codex

- Decision: Logs de diagnostico de libs adicionadas ao `loader.path` devem ser emitidos em nivel DEBUG e consumidos no fluxo com `--debug`.
  Rationale: manter diagnostico detalhado sem poluir execucao padrao.
  Date/Author: 2026-05-13 / User + Codex

- Decision: Em extension mode, `--debug` deve relaxar o override incondicional `logging.level.root=ERROR` e habilitar DEBUG no pacote de bootstrap (`com.seed4j.cli.bootstrap.domain`).
  Rationale: sem esse ajuste, o operador pede diagnostico com `--debug` mas os logs de decisao do milestone 4 continuam ocultos; manter o nivel DEBUG escopado evita ruido global no runtime.
  Date/Author: 2026-05-15 / User + Codex

- Decision: Conflito de versao entre CLI e extensao deve considerar direcao da divergencia (downgrade vs upgrade).
  Rationale: quando o CLI ja possui versao mais nova da mesma coordenada, bloquear bootstrap por fail-fast impede extension mode em casos potencialmente seguros; politica passa a manter o runtime do CLI como vencedor e registrar diagnostico. Quando a extensao exigir versao mais nova que o CLI, o risco de incompatibilidade e mantido como bloqueante.
  Date/Author: 2026-05-13 / User + Codex

- Decision: Conflitos com versoes nao comparaveis devem permanecer bloqueantes com mensagem explicita de causa.
  Rationale: manter fail-fast conservador sem diagnostico explicito torna triagem mais lenta; mensagem dedicada (`not safely comparable`) reduz ambiguidade operacional.
  Date/Author: 2026-05-14 / User + Codex

- Decision: Quando `pom.properties` e nome do arquivo divergem na extensao, a identidade de `pom.properties` prevalece e o bootstrap deve registrar DEBUG explicando o override.
  Rationale: preserva semantica correta por coordenada real e torna auditavel a decisao em cenarios de jar renomeado/shaded.
  Date/Author: 2026-05-14 / User + Codex

- Decision: Materializacao do cache usa staging em `runtime/cache/.<hash>.staging-*` com `move` atomico para `<hash>`.
  Rationale: evita cache parcial em falhas e garante publicacao consistente do overlay.
  Date/Author: 2026-04-30 / Codex

- Decision: Em `extension mode`, publicar `seed4j.cli.runtime.extension.start-class` no bootstrap e consumi-la no child via `spring.main.sources`, mantendo `Seed4JCliApp` como primary source do builder.
  Rationale: habilita descoberta de extensao em pacote customizado sem alterar scan base do CLI e sem duplicar fonte principal.
  Date/Author: 2026-05-04 / Codex

## Risks and Mitigations

- Risk: Remover recursos demais no filtro quebrar `apply` global (readers/templates da extensao deixam de afetar core/extensao).
  Mitigation: filtro por denylist minima (somente recursos globais) + testes de apply em modulo core e modulo da extensao.

- Risk: Override global da extensao alterar versoes/dependencias de modulos do core de forma nao intencional.
  Mitigation: adicionar IT dedicado de sobreposicao explicita (mesma chave/source) + documentar contrato operacional do `apply` em extension mode.

- Risk: Extensao futura depender de lib nova que nao existe no CLI.
  Mitigation: estrategia de libs ausentes com inclusao seletiva e teste dedicado.

- Risk: Permitir conflito nao bloqueante quando a extensao trouxer versao mais antiga pode mascarar dependencia funcional em API removida/mudada da versao mais nova carregada pelo CLI.
  Mitigation: registrar diagnostico detalhado em DEBUG (coordenada + versoes + decisao), manter caminho bloqueante para upgrade da extensao, e incluir validacao manual com extensoes reais para coordenadas sensiveis.

- Risk: `spring.main.sources` incorreto causar duplicidade de beans ou bootstrap inconsistente.
  Mitigation: injetar apenas fonte necessaria da extensao (sem duplicar `Seed4JCliApp`) e cobrir em IT empacotado.

- Risk: Cache corrompido gerar falhas intermitentes.
  Mitigation: escrita atomica + invalidacao por hash + limpeza de staging em erro.

- Risk: Diferencas de OS/path afetarem montagem do `loader.path`.
  Mitigation: usar `Path`/`URI` padrao Java e testes com assercao do comando gerado.

- Risk: Testes empacotados (`PackagedJarIT`) executarem jar stale em `target/` e mascararem regressao/falso negativo no cenario de pacote customizado.
  Mitigation: reconstruir `target/seed4j-cli-*.jar` antes da validacao empacotada e manter comando de validacao do milestone via `failsafe` no fluxo.

- Risk: Heuristica de coordenada/versao por nome de arquivo (`<coordinate>-<version>.jar`) pode gerar falso positivo/negativo para jars fora do padrao.
  Mitigation: adicionar casos de teste para nomes nao convencionais e evoluir a extracao de metadados quando houver evidencia real de conflito nao detectado.

- Risk: Ainda ha ponto de risco pendente no milestone 4: comportamento em extensao real no caminho publico (`seed4j --version`) apos as novas regras de downgrade nao bloqueante + diagnostico DEBUG ainda nao foi revalidado manualmente.
  Mitigation: executar validacao manual com extensao real contendo versao mais antiga e confirmar ausencia de travamento, mensagem de diagnostico e comportamento esperado com `--debug`.

## Validation Strategy

1. Executar testes unitarios focados no bootstrap (`launcher`, `resolver`, `runtime selection`).
2. Executar testes de integracao empacotados para `list`, `apply` (core + extensao com sobreposicao) e `--version` em extension mode.
3. Executar `./mvnw clean verify`.
4. Executar validacao manual de smoke test com `seed4j list` e `seed4j apply` em modulo core e modulo da extensao, em ambiente local de extension mode.

## Rollout and Recovery

Rollout:

1. Publicar versao do `seed4j-cli` com overlay filtrado habilitado por padrao no `extension mode`.
2. Validar em ambiente real com extensao existente (sem regenerar extensao).
3. Monitorar feedback para casos de libs ausentes e para efeitos de override global no `apply`, ajustando testes/documentacao se necessario.

Recovery:

1. Reverter commit(s) da mudanca de bootstrap.
2. Restaurar comportamento anterior de `loader.path` somente se necessario para unblock temporario.
3. Reexecutar suite e smoke tests antes de novo rollout.

## Lessons Learned

- `BOOT-INF/classes` bruto da extensao permite interferencia global mesmo sem `BOOT-INF/lib`.
- Filtro cirurgico de recursos globais preserva contribuicoes de modulo e remove efeito colateral de runtime.
- `loader.path` apontando para overlay local filtrado evita que `config/application*` e `logback*` da extensao alterem o runtime global do CLI.
- O modelo hexagonal reduz acoplamento de dominio, mas nao elimina risco de classpath/config em runtime compartilhado.
- Em `extension mode`, readers/resources de dependencias da extensao atuam globalmente no `apply` por design do contexto Spring compartilhado.
- Para manter `loader.path` robusto, a regra principal e: CLI permanece dono da infraestrutura de runtime, e a extensao pode sobrescrever contribuicoes funcionais de `apply` de forma explicita e testada.
- O milestone 1 pode ser integrado ao launcher sem mudar o `loader.path` imediatamente: materializar cache cedo reduz risco e prepara a troca de path no milestone 2.
- `Start-Class` precisa ser validado em dois niveis: presenca no manifest e existencia do `.class` em `BOOT-INF/classes`, para falha explicita antes de subir o child process.
- `spring.main.sources` com o `Start-Class` da extensao permite carregar contribuicoes fora de `com.seed4j...` sem alterar contrato do gerador nem scan base da aplicacao CLI.
- O fail-fast entre CLI e extensao para coordenada com versao divergente reduz risco de override silencioso, mas ainda depende da qualidade da inferencia de nome do jar.
- A leitura de `META-INF/maven/**/pom.properties` nos jars de `BOOT-INF/lib` reduz falso positivo de conflito entre artefatos homonimos de grupos diferentes; fallback por nome deve permanecer apenas para casos sem metadados.
