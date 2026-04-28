# Seed4J CLI - Investigação de `--version` em extension mode (contexto para novo chat)

Data original da investigação: 2026-04-28  
Atualizado em: 2026-04-28  
Autor: AI agent (este chat)

## Status atual

Este arquivo começou como registro da regressão (`vnull` + warnings de Logback) e foi atualizado com o que **já está resolvido no projeto**.

Com base no ExecPlan [`BUG_EXTENSION_MODE_VERSION_NULL_AND_LOGBACK_WARN_EXEC_PLAN.md`](./BUG_EXTENSION_MODE_VERSION_NULL_AND_LOGBACK_WARN_EXEC_PLAN.md) e no código atual, os dois sintomas principais estão fechados.

## Regressão histórica (estado investigado antes da correção)

Em `seed4j.runtime.mode=extension`, `seed4j --version` chegava a imprimir:

- warnings de Logback sobre arquivos "watchable";
- `Seed4J CLI vnull`;
- `Seed4J version: null`.

O diagnóstico mostrou `resource shadowing` no classpath de extensão (`config/application.yml` e `logback-spring.xml` da extensão aparecendo antes dos do CLI).

## O que já foi resolvido no projeto

1. `Seed4J CLI vnull` deixou de ocorrer.
2. `Seed4J version: null` deixou de ocorrer.
3. Warnings de Logback no `--version` em `extension mode` deixaram de ocorrer.
4. A origem de versão ficou robusta contra shadowing de `project.*`:
   - parent launcher publica `seed4j.cli.version` e `seed4j.cli.seed4j.version`;
   - `Seed4JCommandsFactory` resolve versão por prioridade e fallback explícito (`unknown`).
5. O `logging.config` em `extension mode` ficou determinístico para o CLI:
   - `classpath:seed4j-cli-logback-spring.xml` (nome dedicado, sem ambiguidade com `logback-spring.xml` da extensão).
6. A correção foi blindada com testes e validação final:
   - milestones 1-4 do ExecPlan marcados como concluídos;
   - `./mvnw clean verify` com `BUILD SUCCESS` em 2026-04-28;
   - checkpoint manual com `seed4j --version` em extension mode sem warnings e sem `null` (`EXIT_CODE:0`).

## Evidências objetivas no código atual

- `src/main/java/com/seed4j/cli/bootstrap/domain/Seed4JCliLauncher.java`
  - publica `seed4j.cli.version` e `seed4j.cli.seed4j.version` no child process;
  - usa `logging.config=classpath:seed4j-cli-logback-spring.xml` em `extension mode`.
- `src/main/java/com/seed4j/cli/command/infrastructure/primary/Seed4JCommandsFactory.java`
  - mantém leitura de `project.*` como fallback;
  - prioriza system properties do launcher;
  - aplica fallback final não nulo (`unknown`).
- `src/main/resources/seed4j-cli-logback-spring.xml`
  - arquivo dedicado para baseline de logging do CLI em `extension mode`.

## O que continua válido da investigação (e não foi redesenhado aqui)

1. O modelo de `extension mode` ainda é uma composição de classpath (CLI + extensão), não um isolamento completo.
2. A precedência de recursos/classes da extensão pode continuar influenciando runtime fora do escopo específico do `--version`.
3. Existe risco estrutural se extensão e CLI carregarem dependências-base em versões muito divergentes.

## Evidências estruturais abertas (não resolvidas por este bugfix)

### 6) Verificação de origem real das classes do Spring

Com `-Xlog:class+load=info` em extension mode:

- `org.springframework.boot.SpringApplication` carregada de:
  - `...extension.jar!/BOOT-INF/lib/spring-boot-4.0.1.jar`

Em standard mode:

- `org.springframework.boot.SpringApplication` carregada de:
  - `.../usr/local/bin/seed4j.jar!/BOOT-INF/lib/spring-boot-4.0.6.jar`

Ou seja, extension mode realmente altera quais versões de framework estão em uso.

### 7) Quantificação de conflito de dependências

Comparação de libs (`BOOT-INF/lib`) entre CLI e extensão:

- extensão: 137 jars
- CLI: 139 jars
- artefatos sobrepostos com versão diferente: **75**

Incluindo diferenças em:

- `spring-boot` (`4.0.1` vs `4.0.6`)
- `spring-core` / `spring-context` / `spring-web` etc.
- `logback`
- `jackson`
- `micrometer`
- `tomcat` etc.

### 8) Experimento adicional (só BOOT-INF/lib)

Mesmo removendo `BOOT-INF/classes` do `loader.path` e deixando apenas `BOOT-INF/lib/`, o comportamento `vnull + warnings` continuava.

Conclusão do experimento:

- não era apenas o `application.yml` da própria extensão;
- o `seed4j-2.2.0.jar` da extensão também traz recursos globais (`config/application.yml`, `logback-spring.xml`) e pode prevalecer sobre os do CLI.

## Conclusões estruturais (mantidas)

1. O runtime em extension mode, na investigação feita, **não estava carregando apenas “módulos novos”**.
2. O que ocorria era composição de classpath com runtimes completos (extensão + CLI), com precedência da extensão.
3. Isso afetava diretamente:
   - resolução de recursos (`application.yml`, `logback-spring.xml`),
   - versão efetiva de classes do Spring/infra em runtime.
4. Os sintomas de `null` e warnings eram coerentes com esse modelo de precedência.
5. Há risco estrutural de inconsistência/comportamento divergente enquanto houver mistura de dependências base em versões diferentes no mesmo classpath.

## Pergunta respondida no chat (mantida)

Pergunta do usuário: se versões diferentes de Spring Boot influenciam mesmo, já que a intenção seria pegar só módulos novos.

Resposta técnica baseada nas evidências:

- **Sim, influenciam mesmo.**
- Pela ordem real de classpath, classes centrais do Spring Boot foram carregadas da extensão (4.0.1), não do CLI (4.0.6).
- Portanto, naquele estado investigado, não era um modelo de “somente módulos novos”; era um modelo de sobreposição de runtime.

## Nota para próximos chats

Se o tema for **somente** `--version` com `null` e warnings de Logback em `extension mode`, tratar como **bug já corrigido**.
Se o tema for **isolamento de runtime/compatibilidade entre dependências da extensão e CLI**, isso permanece como discussão arquitetural separada (fora do escopo desta correção).
