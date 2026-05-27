# Extension mode com suporte robusto a bibliotecas Gradle sem `pom.properties`

Este ExecPlan e um documento vivo. Atualize `Progress`, `Decisions`, `Risks` e `Lessons Learned` durante a execucao.

Safety boundary: Esta tarefa e limitada a manutencao defensiva e autorizada deste repositorio para melhorar robustez de bootstrap em `extension mode`. Nao inclui nenhuma orientacao ofensiva.

## Purpose / Big Picture

Hoje o bootstrap de `extension mode` prioriza `META-INF/maven/**/pom.properties` para identidade de bibliotecas e usa fallback por nome de arquivo. Esse contrato funciona para muitos jars Maven, mas nao cobre bem o ecossistema Gradle, onde jars podem nao trazer `pom.properties`. O objetivo deste plano e garantir suporte explicito a jars Gradle (e ecossistema misto Maven+Gradle) sem perder o fail-fast para conflitos reais de runtime. O resultado observavel esperado e: `seed4j list` e `seed4j apply` continuam funcionando em `extension mode` com extensoes empacotadas via Gradle, mesmo quando dependencias aninhadas nao possuem `pom.properties`.

## Scope

Em escopo:

- Endurecer a extracao de identidade de bibliotecas em `BOOT-INF/lib` com estrategia multi-fonte (Maven metadata, Manifest e fallback por nome).
- Remover a dependencia de falha imediata em caso de `pom.properties` incompleto quando houver outras fontes validas de identidade.
- Manter fail-fast para conflitos com evidencia forte de coordenada/versao divergente.
- Registrar diagnostico em nivel DEBUG sobre decisoes de selecao de bibliotecas no `loader.path`.
- Expandir testes unitarios e de integracao de bootstrap para cobrir cenarios Maven-only, Gradle-only e misto.

Fora de escopo:

- Mudar arquitetura para worker JVM separado.
- Alterar contrato do gerador `seed4j-extension`.
- Resolver coordenadas consultando repositorios remotos (Maven Central, Gradle Portal).
- Alterar o comportamento de overlay de `BOOT-INF/classes` fora do necessario para suportar a politica de bibliotecas.

## Definitions

- `biblioteca aninhada`: jar dentro de `BOOT-INF/lib` do fat jar (CLI ou extensao).
- `identidade de runtime`: representacao interna de biblioteca (`coordinate + version`) usada para decidir `present`, `missing` ou `conflict`.
- `evidencia forte`: identidade derivada de metadados estruturados confiaveis (ex.: `pom.properties` completo; manifest com padrao validado de coordenada).
- `evidencia fraca`: identidade inferida de heuristica de nome do arquivo.
- `metadado incompleto`: `pom.properties` presente sem `groupId`, `artifactId` ou `version`.
- `ecossistema misto`: runtime com parte das bibliotecas vindo com metadata Maven e parte sem metadata Maven (comum em build Gradle).

## Existing Context

- O fluxo de montagem de `loader.path` em `extension mode` passa por:
  - `src/main/java/com/seed4j/cli/bootstrap/domain/Seed4JCliLauncher.java`
  - `src/main/java/com/seed4j/cli/bootstrap/domain/RuntimeExtensionLoaderPathResolver.java`
  - `src/main/java/com/seed4j/cli/bootstrap/domain/RuntimeExtensionMissingLibrariesSelector.java`
  - `src/main/java/com/seed4j/cli/bootstrap/domain/RuntimeLibraryIdentity.java`
- O resolver atual le `pom.properties` de jars aninhados e cai para `RuntimeLibraryIdentity.fromJarFileName(...)` quando necessario.
- Ha regra recente de fail-fast para `pom.properties` incompleto em bibliotecas da extensao, adicionada para evitar fallback silencioso em jar renomeado/shaded.

### Study Findings (2026-05-13)

Evidencia pratica coletada com artefatos reais:

1. Fat jar Gradle de exemplo:
   - `/tmp/696302cc-8a7b-4cff-92c4-c87f592b838d/build/libs/seed4j-sample-application-0.0.1-SNAPSHOT.jar`
   - contem `BOOT-INF/lib/*`, `BOOT-INF/classpath.idx`, `BOOT-INF/layers.idx`.
   - nao exibe `META-INF/maven/**/pom.properties` no nivel do jar externo.
2. Jar standalone:
   - `/home/renanfranca/Downloads/micrometer-commons-1.16.5.jar`
   - nao contem `META-INF/maven/**/pom.properties`.
   - manifest contem dados Gradle/Bnd uteis: `Implementation-Title: io.micrometer#micrometer-commons;1.16.5`, `Implementation-Version`, `Bundle-SymbolicName`, `Bundle-Version`, `Automatic-Module-Name`.
3. Jar comum Maven no mesmo runtime:
   - `commons-lang3-3.19.0.jar` contem `META-INF/maven/.../pom.properties`.

Conclusao tecnica: no mesmo `BOOT-INF/lib` podem coexistir bibliotecas com e sem metadata Maven. Tratar `pom.properties` como pre-requisito universal nao e compativel com ecossistema Gradle.

## Desired End State

- Bootstrap de `extension mode` aceita bibliotecas sem `pom.properties` como cenario normal.
- Extracao de identidade usa pipeline de fontes, em ordem de confianca:
  1. `pom.properties` completo (`groupId`, `artifactId`, `version`).
  2. Manifest com padrao valido de coordenada/versao (quando parse for deterministico).
  3. Heuristica por nome de arquivo (`<coordinate>-<version>.jar`) como fallback.
- `pom.properties` incompleto deixa de gerar falha automatica imediata quando existir fonte alternativa; vira evento de diagnostico (DEBUG) e segue para proxima fonte.
- Conflito de versao continua falhando apenas quando houver evidencia forte de mesma coordenada com versao divergente.
- Logs DEBUG mostram quais bibliotecas foram consideradas `present`, `missing` e `conflict`, e por qual fonte de identidade.

## Milestones

### Milestone 1 - Matrizar metadados e contrato de identidade

#### Goal

Tornar explicito, em testes, o comportamento esperado para jars Maven, Gradle e mistos antes de alterar regras de producao.

#### Changes

- [ ] Expandir `src/test/java/com/seed4j/cli/bootstrap/domain/RuntimeExtensionLoaderPathResolverTest.java` com cenarios de metadata:
  - [ ] jar com `pom.properties` completo;
  - [ ] jar sem `pom.properties`, mas com manifest Gradle/Bnd com `Implementation-Title` parseavel;
  - [ ] jar com `pom.properties` incompleto e manifest util;
  - [ ] jar sem metadata Maven e sem manifest parseavel (fallback por nome).
- [ ] Introduzir fixtures de nested jar orientadas a manifest no proprio teste do resolver (ou helper dedicado de fixture em `src/test/java/com/seed4j/cli/bootstrap/domain`).
- [ ] Garantir que os testes expressem comportamento observavel (resultado de `loader.path` ou conflito), nao detalhes internos.

#### Validation

- [ ] Command: `./mvnw -Dtest=RuntimeExtensionLoaderPathResolverTest test`
- [ ] Expected result: casos novos falham em RED com diferencas objetivas do contrato atual (antes da implementacao).

#### Acceptance Criteria

- [ ] Existe cobertura automatizada para os quatro cenarios de metadata definidos.
- [ ] Falhas de RED deixam claro quais fontes de identidade precisam ser suportadas.

### Milestone 2 - Extracao multi-fonte de identidade de biblioteca

#### Goal

Implementar extracao de identidade que suporte jars Maven e Gradle sem assumir `pom.properties` universal.

#### Changes

- [ ] Evoluir `src/main/java/com/seed4j/cli/bootstrap/domain/RuntimeExtensionLoaderPathResolver.java` para usar uma estrategia multi-fonte de identidade.
- [ ] Introduzir tipos dedicados em `src/main/java/com/seed4j/cli/bootstrap/domain` para separar conceito:
  - [ ] fonte de identidade (`MAVEN_POM`, `MANIFEST`, `FILE_NAME`, `NONE`);
  - [ ] nivel de confianca (`STRONG`, `WEAK`, `UNKNOWN`) ou equivalente.
- [ ] Implementar parse de manifest com regra deterministica e documentada (ex.: `Implementation-Title` no formato `group#artifact;version` + consistencia com `Implementation-Version` quando presente).
- [ ] Ajustar politica para `pom.properties` incompleto:
  - [ ] nao falhar imediatamente;
  - [ ] registrar evento diagnostico;
  - [ ] tentar proxima fonte de identidade.

#### Validation

- [ ] Command: `./mvnw -Dtest=RuntimeExtensionLoaderPathResolverTest test`
- [ ] Expected result: cenarios Gradle passam sem regressao dos cenarios Maven existentes.

#### Acceptance Criteria

- [ ] Ausencia de `pom.properties` nao interrompe bootstrap.
- [ ] Identidade derivada por manifest e fallback por nome funciona conforme testes.
- [ ] Contrato de confianca da identidade fica explicito no codigo e nos testes.

### Milestone 3 - Politica de conflito orientada por confianca

#### Goal

Evitar falso positivo de conflito quando a identidade for fraca, sem afrouxar conflito real com evidencia forte.

#### Changes

- [ ] Ajustar `src/main/java/com/seed4j/cli/bootstrap/domain/RuntimeExtensionMissingLibrariesSelector.java` para considerar nivel de confianca da identidade.
- [ ] Manter fail-fast para divergencia de versao quando a coordenada for suportada por evidencia forte.
- [ ] Para identidade fraca/ausente, manter comportamento conservador:
  - [ ] fallback por nome de arquivo apenas quando aplicavel;
  - [ ] conflito explicito quando houver colisao ambigua sem identidade confiavel e risco de override silencioso.
- [ ] Atualizar `src/test/java/com/seed4j/cli/bootstrap/domain/RuntimeExtensionMissingLibrariesSelectorTest.java` para refletir o novo contrato.

#### Validation

- [ ] Command: `./mvnw -Dtest=RuntimeExtensionMissingLibrariesSelectorTest,RuntimeExtensionLoaderPathResolverTest test`
- [ ] Expected result: selecao de `missing` e conflitos permanece deterministica em cenario misto Maven+Gradle.

#### Acceptance Criteria

- [ ] Conflitos fortes continuam falhando com mensagem clara.
- [ ] Cenarios Gradle sem metadata Maven nao quebram por regra excessivamente rigida.
- [ ] Nao ha dependencia de ordem oculta entre extracao e decisao.

### Milestone 4 - Diagnostico DEBUG de bibliotecas selecionadas

#### Goal

Fechar pendencia do milestone 4 original: dar visibilidade operacional das decisoes de libs em `loader.path`.

#### Changes

- [ ] Adicionar logs DEBUG no fluxo de resolucao de bibliotecas em `RuntimeExtensionLoaderPathResolver` (ou componente dedicado), com informacoes:
  - [ ] biblioteca da extensao;
  - [ ] fonte de identidade utilizada;
  - [ ] decisao (`present`, `missing`, `conflict`);
  - [ ] motivo resumido.
- [ ] Garantir que logs sejam silenciosos por padrao e ativaveis explicitamente (ex.: flag/propriedade de runtime).
- [ ] Cobrir ativacao/desativacao em teste unitario quando viavel.

#### Validation

- [ ] Command: `./mvnw -Dtest=Seed4JCliLauncherTest,RuntimeExtensionLoaderPathResolverTest test`
- [ ] Expected result: sem regressao no launcher e logs de diagnostico disponiveis quando DEBUG estiver ativo.

#### Acceptance Criteria

- [ ] Operacao consegue inspecionar por que uma lib entrou (ou nao) no `loader.path`.
- [ ] Fluxo default permanece limpo (sem ruido de log em execucao normal).

### Milestone 5 - Regressao empacotada e alinhamento de documentacao

#### Goal

Validar fim-a-fim o contrato em artefatos empacotados e atualizar documentacao tecnica.

#### Changes

- [ ] Expandir fixtures/ITs de `extension mode` para incluir ao menos um cenario com lib sem `pom.properties` e metadado de manifest Gradle:
  - [ ] `src/test/java/com/seed4j/cli/bootstrap/domain/ExtensionRuntimeFixture.java`
  - [ ] `src/test/java/com/seed4j/cli/bootstrap/domain/ExtensionRuntimeBootstrapPackagedJarIT.java`
  - [ ] `src/test/java/com/seed4j/cli/bootstrap/domain/ExtensionRuntimeBootstrapListPackagedJarIT.java`
- [ ] Atualizar plano principal relacionado:
  - [ ] `_temporary/ai_agent/seed4j-cli-ai-context/shared/2026-04-29_FEATURE_extension-mode-loader-path-filtered-overlay-exec-plan.md`
  - [ ] marcar itens de milestone 4 impactados por suporte Gradle.
- [ ] Documentar a politica final de identidade de bibliotecas em `documentation/` (arquivo a definir durante execucao).

#### Validation

- [ ] Command: `./mvnw -Dtest=RuntimeExtensionMissingLibrariesSelectorTest,RuntimeExtensionLoaderPathResolverTest,Seed4JCliLauncherTest test`
- [ ] Expected result: suites de bootstrap verdes com novos cenarios.
- [ ] Command: `./mvnw -Dit.test=ExtensionRuntimeBootstrapPackagedJarIT,ExtensionRuntimeBootstrapListPackagedJarIT failsafe:integration-test failsafe:verify`
- [ ] Expected result: ITs empacotados verdes no fluxo extension mode.
- [ ] Command: `./mvnw clean verify`
- [ ] Expected result: validacao completa verde (unit, IT, coverage, checkstyle).

#### Acceptance Criteria

- [ ] `extension mode` funciona com bibliotecas Gradle sem `pom.properties`.
- [ ] Politica de conflito continua protegendo contra override silencioso de versao.
- [ ] Diagnostico e documentacao refletem o contrato final.

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

- Decision: Suporte a Gradle sera tratado como requisito de primeira classe no bootstrap de bibliotecas, nao como excecao.
  Rationale: evidencia real de jars em runtime sem `pom.properties` no ecossistema Spring Boot + Gradle.
  Date/Author: 2026-05-13 / User + Codex

- Decision: `pom.properties` incompleto nao deve causar falha imediata sem tentativa de outras fontes de identidade.
  Rationale: evita bloquear runtime valido quando metadados Maven estao ausentes/parciais, cenario comum fora de pipelines Maven puros.
  Date/Author: 2026-05-13 / User + Codex

- Decision: Conflito destrutivo deve continuar exigindo evidencia forte.
  Rationale: reduzir falso positivo mantendo protecao contra override real de versao.
  Date/Author: 2026-05-13 / Codex

## Risks and Mitigations

- Risk: parse de manifest gerar identidade falsa (falso positivo) em jars com campos livres.
  Mitigation: aceitar manifest apenas com padroes deterministas e testes negativos; fallback para nome quando parse for ambiguo.

- Risk: afrouxar demais conflito e permitir override silencioso.
  Mitigation: manter fail-fast em conflitos com evidencia forte e testar cenarios de divergencia real de versao.

- Risk: aumento de complexidade em `RuntimeExtensionLoaderPathResolver`.
  Mitigation: extrair responsabilidade de identidade para tipo dedicado e manter selecao desacoplada da leitura de jar.

- Risk: regressao no tempo de bootstrap por leitura extra de manifest/metadados.
  Mitigation: ler apenas entradas necessarias de nested jar e manter validacao de desempenho em testes de dominio (sem I/O redundante).

- Risk: diagnostico DEBUG vazar ruido em modo normal.
  Mitigation: logs opt-in, default silencioso e cobertura de desativacao.

## Validation Strategy

1. Rodar testes unitarios de identidade e selecao:
   - `./mvnw -Dtest=RuntimeExtensionLoaderPathResolverTest,RuntimeExtensionMissingLibrariesSelectorTest test`
2. Rodar checkpoint vertical do launcher:
   - `./mvnw -Dtest=Seed4JCliLauncherTest#shouldLaunchTheExtensionChildProcessRequestWithLoaderPathAndActiveDistributionSystemProperties test`
3. Rodar ITs empacotados de extension mode:
   - `./mvnw -Dit.test=ExtensionRuntimeBootstrapPackagedJarIT,ExtensionRuntimeBootstrapListPackagedJarIT failsafe:integration-test failsafe:verify`
4. Rodar validacao completa:
   - `./mvnw clean verify`

## Rollout and Recovery

Rollout:

1. Entregar em commits pequenos por milestone (extracao de identidade, politica de conflito, logs, ITs).
2. Validar em CI com foco nas suites de bootstrap e ITs empacotados.
3. Atualizar documentacao operacional do extension mode antes de fechar milestone 5.

Recovery:

1. Reverter milestone corrente se houver regressao (preferencialmente commit unico por milestone).
2. Manter estrategia anterior como fallback temporario apenas se necessario para desbloquear release.
3. Reexecutar testes focados + `clean verify` apos rollback.

## Lessons Learned

- Ecossistema real de `BOOT-INF/lib` e heterogeneo: no mesmo runtime podem coexistir jars com metadata Maven completa e jars sem `pom.properties`.
- Metadados de manifest em artefatos Gradle/Bnd podem oferecer identidade suficiente para reduzir dependencia de heuristica por nome.
- Regra de conflito precisa balancear seguranca (evitar override silencioso) e compatibilidade (nao bloquear jars validos por ausencia de metadata Maven).
- Diagnostico de decisao por biblioteca e essencial para operacao e para depurar divergencias entre ambientes.
