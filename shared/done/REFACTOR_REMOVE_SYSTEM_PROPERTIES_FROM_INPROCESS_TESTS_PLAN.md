# Refatoracao: remover dependencia de System properties globais nos testes in-process

## Resumo

Hoje `ExtensionRuntimeBootstrapInProcessTest` usa `System.setProperty`, `System.clearProperty` e restauracao manual para simular o child-process no mesmo processo JVM. Isso funciona, mas deixa os testes mais frageis por depender de estado global.

Objetivo da refatoracao:

- manter cobertura de comportamento (version + list em extension mode);
- eliminar mutacao de `System` global nesses testes;
- preservar comportamento de producao e dos testes empacotados.

## Estado atual e problema

- O fluxo in-process usa `InProcessChildProcessLauncher` para aplicar `request.systemProperties()` no `System` da JVM antes de rodar o CLI local.
- O provider de runtime no contexto atual (`CurrentProcessRuntimeSelectionProvider`) reconstrui runtime a partir de propriedades do processo atual.
- Consequencia: o teste precisa preparar baseline e restaurar manualmente (`baselineProperties.restore()`), aumentando acoplamento e risco de interferencia entre testes.

## Estrategia de refatoracao (decisao)

Aplicar runtime properties do child request no contexto Spring local, sem tocar `System`, via dois ajustes coordenados:

1. `LocalCliRunner` e `LocalSpringCliRunner` passam a aceitar overrides de runtime properties para uma execucao.
2. Leitura de runtime no comando passa a usar `Environment` (que ja recebe propriedades do builder), em vez de depender exclusivamente de `System.getProperties()`.

Com isso, o teste in-process injeta as propriedades no contexto da execucao e remove toda mutacao global.

## Mudancas tecnicas planejadas

### 1) Expandir contrato do runner local para suportar runtime properties por execucao

- Em `LocalCliRunner`, adicionar sobrecarga/default:
  - `int run(String[] args, Map<String, String> runtimeProperties)`
  - manter `run(String[] args)` delegando para mapa vazio.
- Em `LocalSpringCliRunner`, implementar a nova sobrecarga:
  - aplicar `builder.properties(...)` para cada par `key=value` recebido;
  - manter comportamento atual para `spring.config.location`.

Resultado: o runner local recebe contexto de runtime sem usar `System`.

### 2) Ajustar leitura de runtime selection para considerar Environment

- Trocar o acoplamento em `System.getProperties()` por leitura via `Environment` no caminho atual do comando.
- Garantir que os mesmos tres campos continuem sendo lidos:
  - `seed4j.cli.runtime.mode`
  - `seed4j.cli.runtime.distribution.id`
  - `seed4j.cli.runtime.distribution.version`
- Preservar defaults existentes (`standard` quando modo nao vier).

Resultado: propriedades passadas por `SpringApplicationBuilder.properties(...)` passam a ser suficientes para version/list.

### 3) Reescrever o launcher in-process de teste sem System mutation

- Em `ExtensionRuntimeBootstrapInProcessTest`, `InProcessChildProcessLauncher` deve chamar:
  - `localCliRunner.run(request.arguments().toArray(String[]::new), request.systemProperties())`
- Remover:
  - `System.setProperty(...)`
  - `System.clearProperty(...)`
  - `ScopedSystemProperties` e `baselineProperties.restore()`

Resultado: teste fica isolado por execucao, sem estado global compartilhado.

### 4) Ajustar asserts dos testes in-process

- `shouldExecuteVersionCommandInExtensionModeUsingInProcessChildLauncher`:
  - manter assert de saida com runtime extension + distribution id/version.
- `shouldListExtensionOnlyModuleWhenRunningInExtensionModeUsingInProcessChildLauncher`:
  - manter assert de presenca de `runtime-extension-list-only`.
- Substituir asserts de restauracao global por asserts de comportamento observavel.

## Plano de validacao

Executar na ordem:

1. `./mvnw -Dtest=ExtensionRuntimeBootstrapInProcessTest test`
2. `./mvnw -Dtest=CurrentProcessRuntimeSelectionProviderTest,SystemPropertyRuntimeSelectionProviderTest test`
3. `./mvnw -Dtest=Seed4JCommandsFactoryTest test`
4. `./mvnw -Dit.test=ExtensionRuntimeBootstrapListPackagedJarIT failsafe:integration-test failsafe:verify`
5. `./mvnw clean verify`
6. `npm run prettier:check`

## Riscos e mitigacoes

- Risco: mudanca em leitura de runtime quebrar fluxo atual de version.
  - Mitigacao: manter defaults identicos e validar testes focados de runtime provider.
- Risco: precedence de propriedades no Spring context ficar diferente do esperado.
  - Mitigacao: injetar apenas chaves de runtime nesse caminho e validar saida observavel (`--version` e `list`).
- Risco: regressao no comportamento real (child process empacotado).
  - Mitigacao: manter IT empacotada obrigatoria (`ExtensionRuntimeBootstrapListPackagedJarIT`).

## Criterios de aceite

- Nenhum teste do fluxo in-process usa `System.setProperty`, `System.clearProperty` ou restore manual para runtime selection.
- `ExtensionRuntimeBootstrapInProcessTest` continua cobrindo `--version` e `list` em extension mode.
- `Seed4JCommandsFactoryTest` continua garantindo que standard mode nao lista `runtime-extension-list-only`.
- IT empacotada de extension mode continua verde.
