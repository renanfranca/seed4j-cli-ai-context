# Desenho de Isolamento do Extension Mode (Worker Isolado + Cache)

## Summary

- Trocar o modelo atual de `loader.path` (overlay de `BOOT-INF/classes` e `BOOT-INF/lib`) por isolamento real: o processo principal do CLI mantém runtime/base Spring próprio, e a extensão roda apenas em um **worker JVM separado**.
- Manter comportamento aditivo no catálogo: módulos padrão continuam visíveis; extensão só adiciona contribuições permitidas.
- Preservar performance em uso quente: `--version` e `list` não iniciam worker quando cache já existe; meta de overhead <= `+600ms`.

## Implementation Changes

- **Contrato novo de extensão (cutover direto, sem fallback legado)**:
  - `metadata.yml` passa a exigir `runtime.contract: isolated-worker-v1`.
  - Exigir compatibilidade explícita de versão de core (`compatibility.seed4j-version`) igual ao core do CLI.
  - Exigir no JAR um manifest entry `Seed4J-Extension-Worker-Class` (classe worker gerada pela extensão).
  - Rejeitar extensão sem esse contrato com erro orientando regeneração.

- **Bootstrap do CLI (`extension mode`)**:
  - Remover injeção de `loader.path` da extensão no processo filho principal.
  - Em `extension mode`, antes de subir o filho principal:
    - Se comando for apenas `--version`/`-V`, pular prewarm.
    - Caso contrário, validar contrato e garantir cache por hash do `extension.jar` + `metadata.yml`.
  - Lançar filho principal como runtime padrão do CLI (sem classes/libs da extensão no classpath do CLI).

- **Worker isolado da extensão**:
  - CLI chama worker via `PropertiesLauncher` do próprio `extension.jar` com `-Dloader.main=<worker-class>`.
  - Worker suporta dois comandos:
    - `export-catalog`: exporta catálogo de módulos (slug, descrição, dependências, schema de propriedades).
    - `apply-module`: aplica módulo pelo slug com propriedades resolvidas.
  - Worker sobe contexto da extensão com `web-application-type=none`, lazy init e `spring.config.location` isolado (sem herdar `config/application.yml` da extensão para o runtime do CLI).
  - Política de hidden resources: ignorar hidden vindo da extensão no catálogo importado.

- **Catálogo e comandos no CLI**:
  - Novo cache de catálogo em `~/.config/seed4j-cli/runtime/cache/<hash>/catalog.json`.
  - Merge no CLI:
    - Base padrão do CLI permanece fonte de verdade.
    - Catálogo da extensão contribui apenas slugs adicionais.
    - Se extensão declarar slug já existente no core: falha explícita (sem override silencioso).
  - `apply` em `extension mode`: delegar execução ao worker para **todos** os módulos (decisão tomada), preservando efeito de readers da extensão.
  - Extensão “reader-only” (sem módulos adicionais) deve continuar válida e não alterar/remover catálogo padrão.

- **Mudanças no gerador `seed4j-extension` (repo `seed4j`)**:
  - Gerar classe worker + entry de manifest `Seed4J-Extension-Worker-Class`.
  - Gerar `metadata.yml` no novo formato de contrato.
  - Atualizar documentação de módulo/extensão para o novo fluxo de importação.

## Test Plan

- **Bootstrap/contrato**:
  - Falha para metadata legado (sem `runtime.contract`).
  - Falha para JAR sem worker class no manifest.
  - Falha para `compatibility.seed4j-version` divergente.
  - Assert de que `loader.path` da extensão não é mais enviado ao processo principal.

- **Comportamento funcional**:
  - `--version` em extension mode sem iniciar worker/cache prewarm.
  - `list` preserva catálogo padrão e adiciona apenas módulos da extensão.
  - `list` não remove `seed4j-extension` mesmo se extensão tiver `seed4j.hidden-resources.slugs`.
  - Cenário com pacote `com.mycompany...` funciona (não depende de `com.seed4j...`).
  - `apply` de módulo padrão e módulo da extensão funciona via worker.

- **Performance (aceitação)**:
  - Medir mediana de 5 execuções de `list` e `--version` em modo quente.
  - Critério: overhead de extension mode <= `+600ms` vs standard.
  - Registrar separadamente que primeira execução (cache miss) pode exceder o orçamento.

## Assumptions & Defaults

- Cutover direto confirmado: sem modo legado/fallback.
- Hidden resources da extensão são ignorados no catálogo importado.
- Worker executa `apply` para todos os módulos em extension mode.
- Orçamento de performance aplicado ao caminho quente; cold-start com geração de cache é custo aceito.
