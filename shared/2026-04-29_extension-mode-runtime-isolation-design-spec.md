# Desenho de Isolamento do `extension mode` (CLI dono do runtime)

## Resumo

- Ponto critico confirmado: o modelo atual (`PropertiesLauncher` + `loader.path` com `BOOT-INF/classes` e `BOOT-INF/lib`) mantem risco estrutural de interferencia da extensao no runtime do CLI.
- Adotar fronteira em 2 JVMs no `extension mode`:
1. JVM de comando do CLI (runtime base do `seed4j-cli`, sem classpath da extensao).
2. JVM worker da extensao (controlada pelo CLI, protocolo explicito).
- Corte direto: remover uso de `loader.path` da extensao no processo principal.
- Manter contrato aditivo de catalogo: modulos standard continuam, extensao so adiciona.

## Mudancas de Implementacao

- Bootstrap/runtime:
1. Remover injecao de `loader.path` da extensao no launcher.
2. Passar apenas metadados de runtime e caminho do `extension.jar` por propriedade dedicada (`seed4j.cli.runtime.extension.jar.path`).
3. Preservar baseline de logging do CLI no processo principal.
- Worker + protocolo:
1. Criar modo interno de execucao `extension-worker` no mesmo artefato do CLI.
2. Cliente no CLI chama worker por `stdin/stdout` (JSON) com operacoes `catalog` e `apply`.
3. `catalog` retorna descritores serializaveis de modulos (slug, descricao, propriedades, organizacao/dependencias).
4. `apply` recebe slug + propriedades finais e executa aplicacao no worker.
- Carregamento isolado da extensao no worker:
1. Ler `Start-Class` do `MANIFEST` do `extension.jar` para descobrir pacote base real (suporta `com.seed4j...` e `com.mycompany...`).
2. Carregar `BOOT-INF/classes` e `BOOT-INF/lib` em classloader isolado com parent do CLI (runtime base continua do CLI).
3. Descobrir contribuicoes por capacidade, nao por nome/pacote gerado:
   - `Seed4JModuleResource`
   - beans `Reader` do ecossistema `com.seed4j.module.infrastructure.secondary...`
4. Bloquear influencia de recursos globais da extensao no CLI (`config/application*.yml`, `logback*` nao entram no runtime principal).
- Roteamento de comandos:
1. `list` em `extension mode`: usar catalogo do worker e validar aditividade.
2. `apply` em `extension mode`: delegar execucao ao worker para preservar comportamento de readers gerados pela extensao.
3. `--version` continua local no CLI principal.
4. Falha explicita para slug duplicado entre standard e extensao.

## APIs/Tipos Novos

- `ExtensionWorkerClient` (invocacao de processo e IO de protocolo).
- `ExtensionCatalogItem`, `ExtensionCatalogResponse`.
- `ExtensionApplyRequest`, `ExtensionApplyResponse`.
- `ExtensionRuntimeDescriptor` (jar path, distribuicao, start-class resolvida).
- `ExtensionContributionClassifier` (deteccao de capacidades permitidas).

## Plano de Testes

1. Unitario de launcher: em `extension mode` nao existe `loader.path` no request do processo principal.
2. Unitario de protocolo worker: serializacao/deserializacao de `catalog` e `apply`.
3. Integracao empacotada com `seed4j-sample-extension`: `list` aditivo, `apply` de modulo da extensao funcional.
4. Integracao empacotada com extensao em pacote `com.mycompany...`: descoberta de contribuicoes funciona (sem dependencia de `com.seed4j`).
5. Regressao de `--version`: sem `null` e sem warnings de logback em `extension mode`.
6. Cenarios de falha: `Start-Class` ausente/invalido, slug duplicado, worker indisponivel, `extension.jar` invalido.
7. Validacao completa: `./mvnw clean verify` + checks focados de runtime extension.

## Assuncoes e Defaults

- Sem modo legado/fallback: corte direto conforme decisao.
- Contrato de contribuicao e por capacidade/interface (estavel contra variacao de nomes gerados).
- `metadata.yml` atual permanece; so adicionaremos campos se necessario para diagnostico, nao para ativar o fluxo.
- Objetivo e isolamento de runtime do CLI sem quebrar contribuicoes legitimas de modulo e readers gerados por `seed4j-extension`.
