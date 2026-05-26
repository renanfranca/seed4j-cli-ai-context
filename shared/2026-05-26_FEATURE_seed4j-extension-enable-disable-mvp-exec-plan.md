# Implementar `seed4j extension enable|disable` (MVP sem bypass no launcher)

Este ExecPlan é um documento vivo. Mantenha `Progress`, `Decisions`, `Risks`, e `Lessons Learned` atualizados durante a implementação.

## Purpose / Big Picture

Entregar dois comandos explícitos para alternar o modo de runtime da extensão sem quebrar o contrato já existente de `seed4j extension install`.
O usuário poderá ligar/desligar o modo extension de forma direta, com mensagens claras e comportamento idempotente, mantendo o escopo MVP simples.
Resultado observável: `seed4j extension enable` ativa runtime extension já instalado; `seed4j extension disable` volta para standard sem remover artefatos.

## Scope

In-scope:

- Novo subcomando `seed4j extension enable`.
- Novo subcomando `seed4j extension disable`.
- `seed4j extension install` mantém comportamento atual (instala + ativa extension mode).
- Atualização de help/tests/docs para refletir os três subcomandos.
- `disable` cria `~/.config/seed4j-cli/config.yml` quando ausente com `seed4j.runtime.mode: standard`.

Out-of-scope:

- Bypass do bootstrap/launcher para "modo manutenção".
- Purge de `extension.jar`, `metadata.yml` ou cache em `disable`.
- Mudanças no contrato de validação global do launcher.

Safety boundary: This task is limited to authorized, defensive maintenance of this repository. Do not provide offensive guidance or policy-bypassing instructions.

## Definitions

- `extension mode`: `seed4j.runtime.mode: extension` no `~/.config/seed4j-cli/config.yml`.
- `standard mode`: `seed4j.runtime.mode: standard` no `~/.config/seed4j-cli/config.yml` (ou ausência de chave, mas neste plano `disable` tornará explícito).
- `runtime extension instalado`: presença de `~/.config/seed4j-cli/runtime/active/extension.jar` e `metadata.yml` válidos para bootstrap.
- `MVP sem bypass`: comandos novos não alteram o fluxo de validação inicial do `Seed4JCliLauncher`.

## Existing Context

- `seed4j extension install` já existe e força `mode: extension` com validação de JAR e `config.yml`.
- A árvore de comandos de `extension` hoje só registra `install`.
- O bootstrap falha cedo quando `mode: extension` está inválido; este plano mantém esse comportamento por decisão de escopo MVP.
- Documentação atual (`README.md`, `documentation/Commands.md`) cobre apenas `install` como comando de gestão de extensão.

## Desired End State

- `seed4j extension --help` exibe `install`, `enable` e `disable`.
- `seed4j extension enable`:
- valida runtime extension já instalado (jar/layout + metadata) antes de gravar `mode: extension`;
- retorna sucesso com mensagem objetiva quando válido;
- retorna erro não-zero com mensagem objetiva quando inválido.
- `seed4j extension disable`:
- grava `mode: standard`;
- cria `config.yml` se ausente;
- preserva outras chaves existentes quando YAML é válido;
- não remove artefatos de runtime extension.
- `seed4j extension install` continua com contrato atual (compatibilidade preservada).

## Milestones

### Milestone 1 - Serviço de domínio para alternância de modo runtime

#### Goal

Introduzir tipos/serviços de domínio para habilitar e desabilitar mode com validações e persistência consistentes.

#### Changes

- [ ] Criar serviços de domínio em `src/main/java/com/seed4j/cli/bootstrap/domain` para:
- [ ] habilitar `extension mode` com validação de runtime extension instalado;
- [ ] desabilitar para `standard mode` sem validar artefatos de extensão.
- [ ] Reusar `RuntimeModeConfigReader` para validar e carregar `config.yml`.
- [ ] Reusar `RuntimeSelection.resolve(...)` + `RuntimeExtensionConfiguration.withDefaultPaths(...)` para validação de enable.
- [ ] Gravar `config.yml` preservando demais chaves válidas e alterando apenas `seed4j.runtime.mode`.
- [ ] Garantir criação de `config.yml` em `disable` quando ausente.

#### Validation

- [ ] Command: `./mvnw -Dtest=RuntimeExtensionModeEnablerTest,RuntimeExtensionModeDisablerTest test`
- [ ] Expected result: testes de domínio passam cobrindo sucesso, config inválido, e validação de artefatos no enable.

#### Acceptance Criteria

- [ ] Enable falha com exit de erro quando runtime extension ativo não é válido.
- [ ] Enable grava `mode: extension` quando runtime ativo é válido.
- [ ] Disable grava `mode: standard` e não apaga runtime artifacts.
- [ ] Disable cria `config.yml` quando não existe.
- [ ] Config inválido continua fail-fast (sem autocorreção).

### Milestone 2 - Comandos CLI `extension enable` e `extension disable`

#### Goal

Expor os serviços no CLI com UX alinhada ao padrão de `extension install`.

#### Changes

- [ ] Criar `ExtensionEnableCommand` em `src/main/java/com/seed4j/cli/command/infrastructure/primary`.
- [ ] Criar `ExtensionDisableCommand` em `src/main/java/com/seed4j/cli/command/infrastructure/primary`.
- [ ] Registrar os novos subcomandos em `ExtensionCommand`.
- [ ] Atualizar `CliFixture` (`src/test/java/com/seed4j/cli/command/infrastructure/primary/CliFixture.java`) para montar a árvore com os novos comandos.
- [ ] Ajustar testes de help/árvore de comandos existentes e adicionar testes dedicados para enable/disable.
- [ ] Mensagens de saída padrão:
- [ ] enable sucesso: `Extension runtime enabled successfully.`
- [ ] disable sucesso: `Extension runtime disabled successfully.`

#### Validation

- [ ] Command: `./mvnw -Dtest=ExtensionEnableCommandTest,ExtensionDisableCommandTest,ExtensionInstallCommandTest,Seed4JCommandsFactoryTest test`
- [ ] Expected result: subcomandos aparecem no help e retornam códigos de saída esperados em sucesso/falha.

#### Acceptance Criteria

- [ ] `seed4j extension --help` inclui `install`, `enable`, `disable`.
- [ ] `seed4j extension enable` retorna 0 quando runtime válido e não-zero quando inválido.
- [ ] `seed4j extension disable` retorna 0 e persiste `mode: standard` (inclusive sem config prévio).
- [ ] `seed4j extension install` mantém comportamento atual sem regressão.

### Milestone 3 - Documentação e validação completa

#### Goal

Atualizar contrato público e consolidar comportamento MVP decidido.

#### Changes

- [ ] Atualizar `documentation/Commands.md` com seções de `enable` e `disable`.
- [ ] Atualizar `README.md` para descoberta rápida dos novos comandos.
- [ ] Documentar explicitamente a decisão MVP: sem bypass de launcher; config inválido continua fail-fast.

#### Validation

- [ ] Command: `./mvnw clean verify`
- [ ] Expected result: build, testes, cobertura e checks Java passam.
- [ ] Command: `npm run prettier:check`
- [ ] Expected result: formatação de Markdown/JSON/YAML/Java conforme padrão do repositório.

#### Acceptance Criteria

- [ ] Documentação descreve corretamente os três subcomandos (`install`, `enable`, `disable`).
- [ ] Não há divergência entre comportamento real e contrato documentado.
- [ ] Pipeline local de qualidade passa sem regressão.

## Progress

- [ ] Milestone 1 started
- [ ] Milestone 1 completed
- [ ] Milestone 2 started
- [ ] Milestone 2 completed
- [ ] Milestone 3 started
- [ ] Milestone 3 completed

## Decisions

- Decision: `seed4j extension install` mantém contrato atual de "install + enable".
  Rationale: compatibilidade com uso atual, scripts e documentação.
  Date/Author: 2026-05-26 / Renan + Codex

- Decision: `seed4j extension disable` não remove artefatos de runtime extension.
  Rationale: retorno rápido ao modo extension sem reinstalação.
  Date/Author: 2026-05-26 / Renan + Codex

- Decision: MVP não altera `Seed4JCliLauncher` para bypass de validação.
  Rationale: reduzir complexidade e risco para primeira entrega.
  Date/Author: 2026-05-26 / Renan + Codex

- Decision: se `config.yml` estiver inválido, manter fail-fast com mensagem clara.
  Rationale: manter semântica atual e evitar autocorreção implícita.
  Date/Author: 2026-05-26 / Renan + Codex

- Decision: `disable` cria `config.yml` quando ausente com `mode: standard`.
  Rationale: estado final explícito e previsível.
  Date/Author: 2026-05-26 / Renan + Codex

## Risks and Mitigations

- Risk: usuário esperar que `disable` recupere cenários onde launcher já falha antes de executar comandos.
  Mitigation: documentar limitação do MVP explicitamente em `Commands.md`.

- Risk: divergência de semântica entre validação de enable e bootstrap real.
  Mitigation: usar `RuntimeSelection.resolve(...)` no caminho de enable.

- Risk: regressão na árvore de comandos por mudanças em `ExtensionCommand` e `CliFixture`.
  Mitigation: reforçar testes de help e parse de subcomandos.

- Risk: perda de formatação/comentários no `config.yml`.
  Mitigation: manter comportamento consistente com writer atual; deixar preservação textual fora do escopo MVP.

## Validation Strategy

1. Rodar testes focados de domínio para enable/disable.
2. Rodar testes focados de comandos (`extension` tree + exit codes).
3. Rodar validação completa do repositório com `./mvnw clean verify`.
4. Rodar `npm run prettier:check`.
5. Validar manualmente cenário feliz:
6. `seed4j extension install ...`
7. `seed4j extension disable`
8. `seed4j --version` (espera `Runtime mode: standard`)
9. `seed4j extension enable`
10. `seed4j --version` (espera `Runtime mode: extension`)

## Rollout and Recovery

- Rollout: mudança aditiva via release normal da CLI.
- Recovery técnico:
- se houver regressão nos novos comandos, remover registro de `enable/disable` em `ExtensionCommand` e reverter classes novas;
- manter `install` intacto para preservar capacidade operacional atual.
- Recovery operacional:
- para forçar modo standard em incidente, editar manualmente `~/.config/seed4j-cli/config.yml` para `seed4j.runtime.mode: standard`.

## Lessons Learned

- Limitar MVP a contratos explícitos reduz risco de superdesign no bootstrap.
- Comandos de modo (`enable/disable`) precisam semântica de validação distinta para evitar ambiguidades.
- O principal risco funcional fica na expectativa de "recovery automático"; isso deve ser alinhado por documentação clara.
