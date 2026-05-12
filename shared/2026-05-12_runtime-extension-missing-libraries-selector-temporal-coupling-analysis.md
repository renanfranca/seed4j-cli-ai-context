# RuntimeExtensionMissingLibrariesSelector - Analise de Acoplamento Temporal

## Contexto

Durante o milestone 4, a selecao de `libs ausentes` no `extension mode` foi ajustada para priorizar identidade (`coordenada+versao`) quando disponivel e usar `fileName` apenas como fallback quando a identidade estiver ausente.

Arquivo principal:

- `src/main/java/com/seed4j/cli/bootstrap/domain/RuntimeExtensionMissingLibrariesSelector.java`

## Problema observado

Existe acoplamento temporal entre duas regras de negocio no metodo `select(...)`:

1. `failWhenMissingIdentityShadowsCliLibrary(...)` roda antes e faz fail-fast quando:
   - a lib da extensao nao tem identidade inferivel;
   - e o `fileName` colide com um `fileName` ja presente no CLI.
2. Depois disso, `missingFrom(...)` usa:
   - identidade quando existe;
   - `fileName` no `orElseGet(...)` quando a identidade nao existe.

Trechos relevantes:

- [RuntimeExtensionMissingLibrariesSelector.java:13](/home/renanfranca/projects/seed4j-cli/src/main/java/com/seed4j/cli/bootstrap/domain/RuntimeExtensionMissingLibrariesSelector.java:13)
- [RuntimeExtensionMissingLibrariesSelector.java:69](/home/renanfranca/projects/seed4j-cli/src/main/java/com/seed4j/cli/bootstrap/domain/RuntimeExtensionMissingLibrariesSelector.java:69)
- [RuntimeExtensionMissingLibrariesSelector.java:115](/home/renanfranca/projects/seed4j-cli/src/main/java/com/seed4j/cli/bootstrap/domain/RuntimeExtensionMissingLibrariesSelector.java:115)
- [RuntimeExtensionMissingLibrariesSelector.java:119](/home/renanfranca/projects/seed4j-cli/src/main/java/com/seed4j/cli/bootstrap/domain/RuntimeExtensionMissingLibrariesSelector.java:119)

## Por que isso e acoplamento temporal

O comportamento correto depende da ordem atual:

- primeiro validar/abortar com `failWhenMissingIdentityShadowsCliLibrary(...)`;
- depois calcular `missing` em `missingFrom(...)`.

Se alguem no futuro:

- mover o `failWhenMissingIdentityShadowsCliLibrary(...)` para depois do filtro;
- remover essa validacao por engano;
- ou reusar `missingFrom(...)` isoladamente em outro fluxo;

o sistema pode passar a aceitar/ignorar cenarios que hoje deveriam falhar explicitamente.

Ou seja, a regra de seguranca nao esta totalmente encapsulada no ponto que decide se a lib e `missing`; ela depende de uma pre-condicao de execucao anterior.

## Sintoma em cobertura

No JaCoCo, a linha do `orElseGet(...)` aparece com branch parcial:

- `target/site/jacoco/com.seed4j.cli.bootstrap.domain/RuntimeExtensionMissingLibrariesSelector.java.html`
- linha 119 marcada como `1 of 2 branches missed`.

Interpretacao:

- existe um caminho de branch estruturalmente presente, mas bloqueado no fluxo atual pelo fail-fast anterior;
- isso nao e bug funcional imediato, mas sinaliza regra distribuida em pontos separados com dependencia de ordem.

## Risco de regressao

Risco principal: mudancas de manutencao aparentemente inocentes (reordem, extracao de metodo, reutilizacao parcial) alterarem sem querer a politica de conflito/ausencia de libs.

Impacto possivel:

- conflito que deveria falhar passa a ser silencioso;
- ou lib da extensao deixa de entrar no `loader.path` quando deveria entrar;
- comportamento divergente entre intencao do milestone e runtime real.

## Direcoes de correcao

### Opcao A (minima, baixo impacto)

Manter estrutura atual, mas tornar a dependencia de ordem explicita:

- documentar pre-condicao perto de `missingFrom(...)`;
- reforcar testes que provam o contrato de ordem no `select(...)`;
- evitar expor/reusar `missingFrom(...)` fora desse fluxo.

Vantagem:

- menor risco de alterar comportamento agora.

Limite:

- acoplamento temporal continua existindo.

### Opcao B (mais robusta, recomendada)

Centralizar decisao por biblioteca em um unico ponto:

- exemplo conceitual: `decisionFor(extensionLibrary, cliRuntimeLibraries)` retornando
  - `MISSING`,
  - `PRESENT`,
  - `CONFLICT`.
- `select(...)` apenas agrega `MISSING` e interrompe em `CONFLICT`.

Vantagens:

- elimina dependencia implicita de ordem;
- reduz branch inalcançavel por contrato externo;
- facilita manutencao e leitura da regra.

Trade-off:

- exige pequeno refactor e ajuste de testes.

## Estado atual para retomada

- comportamento funcional atual segue verde nos testes relevantes executados na conversa;
- o ponto em aberto e de robustez de design (acoplamento temporal), nao de falha funcional imediata.

Sugestao para retomar no proximo chat:

1. decidir entre Opcao A e B;
2. abrir ciclo TDD pequeno focado apenas nessa regra;
3. validar novamente `RuntimeExtensionMissingLibrariesSelectorTest` e `RuntimeExtensionLoaderPathResolverTest`.
