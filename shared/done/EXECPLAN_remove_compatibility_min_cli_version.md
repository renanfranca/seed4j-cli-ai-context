# Remover `compatibility.min-cli-version` com ignorar silencioso e limpeza do fluxo de runtime

Este ExecPlan é um documento vivo. Atualize `Progress`, `Decisions`, `Risks` e `Lessons Learned` durante a execução.

## Purpose / Big Picture

Vamos remover o contrato de `compatibility.min-cli-version` do `metadata.yml` sem quebrar metadata legado: qualquer conteúdo de `compatibility` será ignorado silenciosamente.
O resultado observável é que o runtime extension continuará funcionando com ou sem `compatibility`, inclusive quando essa seção estiver malformada, e o código de comparação de versão de CLI para esse fluxo será eliminado.

Safety boundary: This task is limited to authorized, defensive maintenance of this repository. Do not provide offensive guidance or policy-bypassing instructions.

## Scope

In-scope:

- Remover parsing/validação/uso de `compatibility.min-cli-version` no bootstrap de runtime extension.
- Remover limpeza técnica associada: assinatura de `RuntimeSelection.resolve(...)` sem `currentCliVersion` e remoção de `CliVersion`.
- Atualizar testes unitários/fixture impactados.
- Atualizar `documentation/Commands.md` removendo menções ao campo legado, sem nota de migração.

Out-of-scope:

- Alterar publicação da system property `seed4j.cli.version` no child process.
- Mudanças de arquitetura fora do fluxo `bootstrap/domain` de seleção de runtime.
- Atualizações em arquivos de contexto histórico em `_temporary/...`.

## Definitions

- `metadata.yml`: arquivo de metadados do runtime extension em `~/.config/seed4j-cli/runtime/active/metadata.yml`.
- `Ignorar silenciosamente`: aceitar o campo sem erro, sem warning e sem efeito funcional.
- `Seleção de runtime`: decisão entre `STANDARD` e `EXTENSION` feita por `RuntimeSelection`.

## Existing Context

- Hoje `RuntimeMetadata` carrega `distribution.*` e opcionalmente `compatibility.min-cli-version`.
- `RuntimeSelection.resolve(runtimeConfiguration, currentCliVersion)` valida compatibilidade usando `CliVersion`.
- `RuntimeSelectionTest` cobre múltiplos cenários de bloqueio/parse de `min-cli-version`.
- `ExtensionRuntimeFixtureTest` também chama `RuntimeSelection.resolve(...)` com versão da CLI.
- `documentation/Commands.md` ainda documenta `compatibility.min-cli-version` como opcional e lista falhas relacionadas.

## Desired End State

- `metadata.yml` terá contrato efetivo somente com `distribution.id` e `distribution.version`.
- Qualquer conteúdo em `compatibility` será ignorado (incluindo tipo inválido).
- `RuntimeSelection.resolve(...)` não receberá mais `currentCliVersion`.
- `CliVersion.java` será removido por código morto nesse fluxo.
- Testes refletirão o novo contrato.
- Documentação oficial não citará mais `compatibility.min-cli-version`.

## Milestones

### Milestone 1 - Remover contrato de compatibilidade do código de produção

#### Goal

Eliminar a lógica funcional de `min-cli-version` e simplificar a API de seleção de runtime.

#### Changes

- [ ] Editar `src/main/java/com/seed4j/cli/bootstrap/domain/RuntimeMetadata.java` para:
- [ ] Remover o campo `compatibilityMinCliVersion` do record.
- [ ] Ler apenas `distribution.id` e `distribution.version`.
- [ ] Ignorar completamente `compatibility` quando presente.
- [ ] Remover helpers privados que ficarem sem uso após a simplificação.
- [ ] Editar `src/main/java/com/seed4j/cli/bootstrap/domain/RuntimeSelection.java` para:
- [ ] Alterar assinatura para `resolve(RuntimeConfiguration runtimeConfiguration)`.
- [ ] Remover chamada de validação via `CliVersion`.
- [ ] Editar `src/main/java/com/seed4j/cli/bootstrap/domain/Seed4JCliLauncher.java` para chamar a nova assinatura de `RuntimeSelection.resolve(...)`.
- [ ] Remover `src/main/java/com/seed4j/cli/bootstrap/domain/CliVersion.java`.

#### Validation

- [ ] Command: `rg -n "compatibilityMinCliVersion|minimumCompatibility|validateCompatibilityWith|CliVersion\\." src/main/java`
- [ ] Expected result: nenhuma referência restante no código de produção.
- [ ] Command: `./mvnw -DskipTests compile`
- [ ] Expected result: compilação do código principal concluída sem erro.

#### Acceptance Criteria

- [ ] `RuntimeSelection` não depende mais de versão da CLI para resolver runtime extension.
- [ ] `CliVersion` removido sem referências órfãs.

### Milestone 2 - Atualizar testes para o novo contrato (ignorar silenciosamente)

#### Goal

Fixar o comportamento novo em testes e remover cenários que não existem mais.

#### Changes

- [ ] Editar `src/test/java/com/seed4j/cli/bootstrap/domain/RuntimeSelectionTest.java` para:
- [ ] Atualizar chamadas para `RuntimeSelection.resolve(runtimeConfiguration)` sem versão.
- [ ] Remover testes de aceitação/rejeição por `min-cli-version` e de parse de versão da CLI nesse contexto.
- [ ] Remover cenários que esperavam erro para `compatibility` inválido.
- [ ] Adicionar testes que provem ignore silencioso, incluindo `compatibility` malformado (ex.: string em vez de mapa).
- [ ] Manter cobertura de validação obrigatória de `distribution.id` e `distribution.version`.
- [ ] Editar `src/test/java/com/seed4j/cli/bootstrap/domain/ExtensionRuntimeFixtureTest.java` para nova assinatura de `resolve(...)`.
- [ ] Ajustar quaisquer outros testes quebrados apenas por mudança de assinatura.

#### Validation

- [ ] Command: `./mvnw -Dtest=RuntimeSelectionTest,ExtensionRuntimeFixtureTest test`
- [ ] Expected result: classes passam e incluem evidência de que `compatibility` não interfere.
- [ ] Command: `rg -n "RuntimeSelection\\.resolve\\([^\\)]*,\\s*\"|RuntimeSelection\\.resolve\\([^\\)]*,\\s*currentCliVersion" src/test/java src/main/java`
- [ ] Expected result: nenhuma chamada com segundo parâmetro de versão.

#### Acceptance Criteria

- [ ] Existe teste explícito mostrando `compatibility` inválido sendo ignorado sem exceção.
- [ ] Não há testes remanescentes exigindo validação de `min-cli-version`.

### Milestone 3 - Atualizar documentação oficial e validar o projeto inteiro

#### Goal

Alinhar contrato documentado com o comportamento implementado e fechar validação completa.

#### Changes

- [ ] Editar `documentation/Commands.md` para:
- [ ] Remover `compatibility.min-cli-version` dos exemplos de `metadata.yml`.
- [ ] Remover regras e failure cases relacionados a esse campo.
- [ ] Manter foco no contrato obrigatório de `distribution.id` e `distribution.version`.
- [ ] Não adicionar nota de migração (decisão explícita).

#### Validation

- [ ] Command: `./mvnw clean verify`
- [ ] Expected result: suíte completa (testes, checkstyle, cobertura) verde.
- [ ] Command: `npm run prettier:check`
- [ ] Expected result: sem violações de formatação em Markdown e demais arquivos suportados.
- [ ] Command: `rg -n "min-cli-version|compatibility\\.min-cli-version" src/main src/test documentation README.md`
- [ ] Expected result: sem ocorrências em código/documentação oficial.

#### Acceptance Criteria

- [ ] Build completo passa.
- [ ] Documentação oficial não cita mais o campo removido.

## Progress

- [x] Milestone 1 started
- [x] Milestone 1 completed
- [x] Milestone 2 started
- [x] Milestone 2 completed
- [x] Milestone 3 started
- [x] Milestone 3 completed

## Decisions

- Decision: `compatibility` inteiro será ignorado silenciosamente, independentemente de tipo/estrutura.
  Rationale: decisão explícita do usuário para não quebrar metadata legado e evitar ruído operacional.
  Date/Author: 2026-05-25 / Renan + Codex

- Decision: remover `currentCliVersion` da assinatura de `RuntimeSelection.resolve(...)`.
  Rationale: após remover validação de compatibilidade, esse parâmetro não influencia mais seleção de runtime.
  Date/Author: 2026-05-25 / Renan + Codex

- Decision: remover menções ao campo na documentação sem nota de migração.
  Rationale: decisão explícita do usuário para simplificar documentação e evitar seção transitória.
  Date/Author: 2026-05-25 / Renan + Codex

## Risks and Mitigations

- Risk: configuração inválida em `compatibility` deixar de ser detectada e passar despercebida.
  Mitigation: risco aceito por decisão; adicionar testes explícitos para garantir comportamento consistente de ignore silencioso.

- Risk: mudança de assinatura causar quebras de compilação em testes auxiliares.
  Mitigation: busca global por `RuntimeSelection.resolve(` e validação incremental com testes focados + compile.

- Risk: remoção de documentação sem nota gerar dúvida para quem usava campo legado.
  Mitigation: manter exemplos mínimos corretos e contrato objetivo baseado apenas em `distribution.*`.

- Risk: `npm run prettier:check` falhar por arquivos históricos fora do escopo em `_temporary/...`.
  Mitigation: formatar apenas arquivos alterados neste trabalho e registrar que a falha residual é baseline pré-existente.

## Validation Strategy

1. Validar remoção de referências e compilar produção (`-DskipTests compile`) após Milestone 1.
2. Rodar testes focados de seleção de runtime e fixture após Milestone 2.
3. Rodar `./mvnw clean verify` e `npm run prettier:check` como gate final.
4. Confirmar por `rg` que não restam referências oficiais ao campo removido.

## Rollout and Recovery

Rollout:

1. Entregar em commit único de refactor/fix focado no contrato de metadata extension.
2. Abrir PR com evidências dos comandos de validação e resumo da mudança de contrato.

Recovery:

1. Se surgir regressão funcional, reverter commit completo para restaurar comportamento anterior rapidamente.
2. Reaplicar em passos menores (produção -> testes -> docs) se o problema estiver em cobertura incompleta.

## Lessons Learned

- A decisão “ignorar silenciosamente” conflita com “sem retrocompatibilidade”; a prioridade final foi compatibilidade prática.
- Sem limpeza de assinatura, o código manteria acoplamento artificial com versão da CLI.
- Fixar o novo contrato em testes de comportamento evita reintrodução acidental da validação de `compatibility`.
- `npm run prettier:check` usa escopo amplo e inclui `_temporary/...`; como esses arquivos não eram parte do plano, a validação residual não representa regressão da mudança implementada.
