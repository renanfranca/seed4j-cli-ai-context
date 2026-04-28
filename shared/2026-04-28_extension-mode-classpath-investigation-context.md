# Seed4J CLI - Investigação de `--version` em extension mode (contexto para novo chat)

Data: 2026-04-28
Autor: AI agent (este chat)

## Status desta investigação

Este documento registra **as evidências e conclusões levantadas neste chat** durante a investigação.

Observação importante: após isso, o usuário informou que já resolveu os sintomas de `null/warnings`.

---

## Sintoma inicial observado

Com `seed4j.runtime.mode=extension`, ao executar:

```bash
seed4j --version
```

a saída estava assim:

```text
|-WARN ... Missing watchable .xml or .properties files.
|-WARN ... Watching .xml files requires that the main configuration file is reachable as a URL
Seed4J CLI vnull
Seed4J version: null
Runtime mode: extension
Distribution ID: sample-extension
Distribution version: 1.0.0
```

Com `standard mode` (ex.: `-Duser.home=/tmp/seed4j-noext`), a versão vinha correta (`0.0.1-SNAPSHOT` / `2.2.0`).

---

## Evidências coletadas

## 1) Como o CLI é iniciado

`/usr/local/bin/seed4j`:

```bash
java -jar "/usr/local/bin/seed4j.jar" "$@"
```

## 2) Pontos de código que explicam o comportamento

- `src/main/java/com/seed4j/cli/bootstrap/domain/Seed4JCliLauncher.java`
  - Em extension mode injeta propriedades de child process, incluindo:
    - `-Dloader.path=...`
    - `-Dlogging.config=classpath:logback-spring.xml`
- `src/main/java/com/seed4j/cli/bootstrap/domain/RuntimeExtensionLoaderPathResolver.java`
  - Resolve para:
    - `jar:<extension.jar>!/BOOT-INF/classes`
    - `jar:<extension.jar>!/BOOT-INF/lib/`
- `src/main/java/com/seed4j/cli/command/infrastructure/primary/Seed4JCommandsFactory.java`
  - `@Value("${project.version}")` e `@Value("${project.seed4j-version}")`
- `src/main/resources/config/application.yml`
  - Define `project.version` e `project.seed4j-version`.

## 3) Recurso de extensão sobrescrevendo recurso do CLI

Dentro de `~/.config/seed4j-cli/runtime/active/extension.jar` foram encontrados:

- `BOOT-INF/classes/config/application.yml`
- `BOOT-INF/classes/logback-spring.xml`
- `BOOT-INF/lib/seed4j-2.2.0.jar` (também contendo `config/application.yml` e `logback-spring.xml`)

O `logback-spring.xml` da extensão/seed4j tinha `scan="true"`.

## 4) Reprodução controlada do problema

Executando o `PropertiesLauncher` com o mesmo `loader.path` do launcher:

```bash
java \
  -Dseed4j.cli.runtime.child=true \
  -Dseed4j.cli.runtime.mode=extension \
  -Dseed4j.cli.runtime.distribution.id=sample-extension \
  -Dseed4j.cli.runtime.distribution.version=1.0.0 \
  -Dloader.path=jar:file:/home/renanfranca/.config/seed4j-cli/runtime/active/extension.jar!/BOOT-INF/classes,jar:file:/home/renanfranca/.config/seed4j-cli/runtime/active/extension.jar!/BOOT-INF/lib/ \
  -Dlogging.config=classpath:logback-spring.xml \
  -Dlogging.level.root=ERROR \
  -Dspring.main.log-startup-info=false \
  -cp /usr/local/bin/seed4j.jar \
  org.springframework.boot.loader.launch.PropertiesLauncher --version
```

Resultado: reproduziu exatamente warnings + `vnull`.

## 5) Confirmação de precedência de classpath

Com `loader.debug=true`, o launcher mostrou:

- entries da extensão adicionadas **antes** do root archive (`seed4j.jar`).

Também foi feito um `ResourceProbe` para listar recursos resolvidos:

- `config/application.yml`
  - 1º da extensão
  - depois CLI
- `logback-spring.xml`
  - 1º da extensão
  - depois CLI

## 6) Verificação de origem real das classes do Spring

Com `-Xlog:class+load=info` em extension mode:

- `org.springframework.boot.SpringApplication` carregada de:
  - `...extension.jar!/BOOT-INF/lib/spring-boot-4.0.1.jar`

Em standard mode:

- `org.springframework.boot.SpringApplication` carregada de:
  - `.../usr/local/bin/seed4j.jar!/BOOT-INF/lib/spring-boot-4.0.6.jar`

Ou seja, extension mode realmente altera quais versões de framework estão em uso.

## 7) Quantificação de conflito de dependências

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

## 8) Experimento adicional (só BOOT-INF/lib)

Mesmo removendo `BOOT-INF/classes` do `loader.path` e deixando apenas `BOOT-INF/lib/`, o comportamento `vnull + warnings` continuava.

Conclusão do experimento:

- não era apenas o `application.yml` da própria extensão;
- o `seed4j-2.2.0.jar` da extensão também traz recursos globais (`config/application.yml`, `logback-spring.xml`) e pode prevalecer sobre os do CLI.

---

## Conclusões (as mesmas reportadas no chat)

1. O runtime em extension mode, na investigação feita, **não estava carregando apenas “módulos novos”**.
2. O que ocorria era composição de classpath com runtimes completos (extensão + CLI), com precedência da extensão.
3. Isso afetava diretamente:
   - resolução de recursos (`application.yml`, `logback-spring.xml`),
   - versão efetiva de classes do Spring/infra em runtime.
4. Os sintomas de `null` e warnings eram coerentes com esse modelo de precedência.
5. Há risco estrutural de inconsistência/comportamento divergente enquanto houver mistura de dependências base em versões diferentes no mesmo classpath.

---

## Pergunta respondida no chat

Pergunta do usuário: se versões diferentes de Spring Boot influenciam mesmo, já que a intenção seria pegar só módulos novos.

Resposta técnica baseada nas evidências:

- **Sim, influenciam mesmo.**
- Pela ordem real de classpath, classes centrais do Spring Boot foram carregadas da extensão (4.0.1), não do CLI (4.0.6).
- Portanto, naquele estado investigado, não era um modelo de “somente módulos novos”; era um modelo de sobreposição de runtime.

---

## Nota final

Este arquivo registra o estado e as conclusões da investigação neste chat, antes da correção local aplicada depois pelo usuário.
