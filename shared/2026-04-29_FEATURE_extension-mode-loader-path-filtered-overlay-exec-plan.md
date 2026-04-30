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

- [ ] Implementar filtro de recursos globais no overlay:
  - [ ] remover `config/application*.yml`;
  - [ ] remover `config/application*.yaml`;
  - [ ] remover `config/application*.properties`;
  - [ ] remover `logback*.xml`.
- [ ] Preservar recursos funcionais necessarios (ex.: `generator/**`, `messages/**`, templates e assets), incluindo os usados por readers compartilhados de dependencias.
- [ ] Atualizar `RuntimeExtensionLoaderPathResolver` para receber caminho do overlay local em vez de URL `jar:` para `BOOT-INF/classes`.
- [ ] Manter baseline de logging do CLI no child process.

#### Validation

- [ ] Command: `./mvnw -Dtest=Seed4JCliLauncherTest,ExtensionRuntimeBootstrapInProcessTest test`
- [ ] Expected result: `loader.path` aponta para overlay local filtrado e saida continua limpa.

#### Acceptance Criteria

- [ ] `seed4j.hidden-resources` da extensao nao remove mais modulo do core em `list`.
- [ ] Modulos da extensao continuam sendo descobertos.
- [ ] Recursos usados por readers de dependencias da extensao permanecem acessiveis no overlay.

### Milestone 3 - Descoberta robusta para pacote customizado (`Start-Class`)

#### Goal

Garantir que contribuicoes da extensao sejam carregadas mesmo quando a extensao usa pacote base customizado (ex.: `com.mycompany...`).

#### Changes

- [ ] Ler `Start-Class` do manifest do `extension.jar` durante bootstrap.
- [ ] Publicar propriedade de runtime dedicada com `start-class` resolvido.
- [ ] Ajustar inicializacao Spring no child path para incluir `spring.main.sources` apropriado sem duplicar `Seed4JCliApp`.
- [ ] Cobrir cenarios:
  - [ ] extensao `com.seed4j...`;
  - [ ] extensao `com.mycompany...`.

#### Validation

- [ ] Command: `./mvnw -Dtest=ExtensionRuntimeBootstrapListPackagedJarIT,ExtensionRuntimeBootstrapPackagedJarIT failsafe:integration-test failsafe:verify`
- [ ] Expected result: list/apply em extension mode funcionam para pacotes diferentes sem erro de bean duplicado, mantendo override global de readers/resources da extensao.

#### Acceptance Criteria

- [ ] O fluxo nao depende de naming fixo `com.seed4j.extension`.
- [ ] `Start-Class` ausente/invalido falha com mensagem explicita.
- [ ] O comportamento global de override de readers/resources independe do pacote base da extensao.

### Milestone 4 - Politica de `BOOT-INF/lib` sem sobrescrever runtime do CLI

#### Goal

Definir e implementar politica segura para bibliotecas da extensao mantendo `loader.path` robusto.

#### Changes

- [ ] Implementar estrategia default: nao importar `BOOT-INF/lib` da extensao quando todos jars ja existem no CLI.
- [ ] Implementar estrategia controlada para `libs ausentes`:
  - [ ] detectar jars da extensao nao presentes no CLI;
  - [ ] adicionar somente esses jars ao `loader.path` (sem trocar versoes base).
- [ ] Adicionar validacao/fail-fast para conflitos de coordenada com versao divergente quando detectado risco de override.
- [ ] Registrar decisao em runtime logs de diagnostico (nivel DEBUG) sobre quais libs foram efetivamente adicionadas.

#### Validation

- [ ] Command: `./mvnw -Dtest=RuntimeSelectionTest,Seed4JCliLauncherTest,ExtensionRuntimeBootstrapInProcessTest test`
- [ ] Expected result: casos com libs equivalentes continuam verdes; casos com libs ausentes exercitam adicionamento seletivo.

#### Acceptance Criteria

- [ ] O classpath do CLI permanece fonte principal da infraestrutura.
- [ ] Extensao so adiciona libs quando realmente ausentes.

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
- [ ] Milestone 2 started
- [ ] Milestone 2 completed
- [ ] Milestone 3 started
- [ ] Milestone 3 completed
- [ ] Milestone 4 started
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

- Decision: Materializacao do cache usa staging em `runtime/cache/.<hash>.staging-*` com `move` atomico para `<hash>`.
  Rationale: evita cache parcial em falhas e garante publicacao consistente do overlay.
  Date/Author: 2026-04-30 / Codex

## Risks and Mitigations

- Risk: Remover recursos demais no filtro quebrar `apply` global (readers/templates da extensao deixam de afetar core/extensao).
  Mitigation: filtro por denylist minima (somente recursos globais) + testes de apply em modulo core e modulo da extensao.

- Risk: Override global da extensao alterar versoes/dependencias de modulos do core de forma nao intencional.
  Mitigation: adicionar IT dedicado de sobreposicao explicita (mesma chave/source) + documentar contrato operacional do `apply` em extension mode.

- Risk: Extensao futura depender de lib nova que nao existe no CLI.
  Mitigation: estrategia de libs ausentes com inclusao seletiva e teste dedicado.

- Risk: `spring.main.sources` incorreto causar duplicidade de beans ou bootstrap inconsistente.
  Mitigation: injetar apenas fonte necessaria da extensao (sem duplicar `Seed4JCliApp`) e cobrir em IT empacotado.

- Risk: Cache corrompido gerar falhas intermitentes.
  Mitigation: escrita atomica + invalidacao por hash + limpeza de staging em erro.

- Risk: Diferencas de OS/path afetarem montagem do `loader.path`.
  Mitigation: usar `Path`/`URI` padrao Java e testes com assercao do comando gerado.

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
- O modelo hexagonal reduz acoplamento de dominio, mas nao elimina risco de classpath/config em runtime compartilhado.
- Em `extension mode`, readers/resources de dependencias da extensao atuam globalmente no `apply` por design do contexto Spring compartilhado.
- Para manter `loader.path` robusto, a regra principal e: CLI permanece dono da infraestrutura de runtime, e a extensao pode sobrescrever contribuicoes funcionais de `apply` de forma explicita e testada.
- O milestone 1 pode ser integrado ao launcher sem mudar o `loader.path` imediatamente: materializar cache cedo reduz risco e prepara a troca de path no milestone 2.
