# Automatizar instalação de extensão ativa (`seed4j extension install`) com habilitação de runtime extension

Este ExecPlan é um documento vivo. Mantenha `Progress`, `Decisions`, `Risks`, e `Lessons Learned` atualizados durante a implementação.

## Purpose / Big Picture

Hoje o fluxo de extensão depende de passos manuais (`cp` do JAR, manutenção de `metadata.yml` e edição de `config.yml`). Isso gera erro operacional e inconsistência de ambiente.  
O objetivo é entregar um comando único que automatize esse fluxo mínimo com fail-fast seguro: instalar/sobrescrever a extensão ativa, gravar metadados de distribuição e habilitar `seed4j.runtime.mode: extension`.  
O resultado observável será executar `seed4j extension install ...` com sucesso e, em seguida, validar com `seed4j --version` e `seed4j list`.

## Scope

In-scope:
- Novo comando CLI: `seed4j extension install <jar> --distribution-id <id> --distribution-version <version>`.
- Sobrescrita da extensão ativa sem `--force`.
- Criação de `~/.config/seed4j-cli/config.yml` quando não existir.
- Falha explícita se `config.yml` existir e for inválido.
- Atualização de documentação do fluxo de extensão.

Out-of-scope:
- Repositório de múltiplas versões de extensão.
- Download remoto de artefatos.
- Assinatura/verificação criptográfica de JAR.
- Semântica de rollback completo entre versões de extensão.

Safety boundary: This task is limited to authorized, defensive maintenance of this repository. Do not provide offensive guidance or policy-bypassing instructions.

## Definitions

- `runtime ativo`: par de arquivos consumidos pelo bootstrap em `~/.config/seed4j-cli/runtime/active/` (`extension.jar` e `metadata.yml`).
- `metadata.yml`: arquivo YAML com `distribution.id` e `distribution.version`.
- `config.yml`: `~/.config/seed4j-cli/config.yml`, onde `seed4j.runtime.mode` define `standard|extension`.
- `fail-fast`: erro interrompe execução com exit code não-zero e mensagem clara.
- `instalação idempotente`: rodar o mesmo comando novamente não corrompe estado.

## Existing Context

- O contrato manual atual está documentado em `documentation/Commands.md` (seção de Extension Runtime Metadata).
- O bootstrap já valida JAR de extensão (`BOOT-INF/classes`) e metadata em `com.seed4j.cli.bootstrap.domain`.
- A árvore de comandos Picocli é montada por `Seed4JCommandsFactory`, consumindo `Seed4JCommand`.
- Não existe hoje comando `extension` na camada `command/infrastructure/primary`.

## Desired End State

- Usuário executa:
  `seed4j extension install my-extension-1-0-0.jar --distribution-id my-extension --distribution-version 1.0.0`
- CLI valida o JAR e configuração existente.
- CLI sobrescreve `runtime/active/extension.jar` e `runtime/active/metadata.yml`.
- CLI cria/atualiza `config.yml` para `seed4j.runtime.mode: extension`.
- Em sucesso, CLI imprime confirmação e sempre sugere:
  `seed4j --version`
  `seed4j list`
- Em erro de `config.yml` inválido, falha antes de mutar runtime/config.

## Milestones

### Milestone 1 - Modelagem de domínio e serviço de instalação

#### Goal

Implementar núcleo de instalação com validação e escrita segura, separado da camada de comando.

#### Changes

- Criar serviço de domínio em `src/main/java/com/seed4j/cli/bootstrap/domain` para instalar extensão ativa a partir de um request tipado.
- Introduzir tipos dedicados para conceitos de negócio mínimos:
  - request de instalação (jar + distribution id/version),
  - resultado de instalação (caminhos efetivos + indicador de sobrescrita).
- Reusar validação de layout de runtime extension (JAR com `BOOT-INF/classes`).
- Validar `config.yml` existente com mesma semântica já usada no bootstrap; se inválido, lançar erro claro e abortar.
- Se `config.yml` não existir, criar arquivo mínimo com `seed4j.runtime.mode: extension`.
- Se `config.yml` existir e for válido, preservar demais chaves e apenas garantir `seed4j.runtime.mode: extension`.
- Gravar `extension.jar` e `metadata.yml` com estratégia segura (temp + move com replace).

#### Validation

- `./mvnw -Dtest=RuntimeExtensionInstallerTest test`
- `./mvnw -Dtest=RuntimeSelectionTest test`

#### Acceptance Criteria

- Serviço falha quando JAR não é runtime extension válido.
- Serviço falha quando `config.yml` existente é inválido.
- Serviço cria `config.yml` quando ausente.
- Serviço sempre deixa `seed4j.runtime.mode: extension` no estado final de sucesso.
- Serviço sobrescreve runtime ativo existente sem exigir `--force`.

### Milestone 2 - Comando CLI `extension install`

#### Goal

Expor o fluxo via Picocli no padrão do projeto.

#### Changes

- Adicionar comando raiz `extension` na camada `src/main/java/com/seed4j/cli/command/infrastructure/primary`.
- Adicionar subcomando `install` com:
  - argumento posicional `<jar>`,
  - opções obrigatórias `--distribution-id` e `--distribution-version`.
- Integrar subcomando ao serviço de domínio do Milestone 1.
- Definir comportamento de saída:
  - sucesso: confirmação da instalação e aviso de validação com `seed4j --version` e `seed4j list`,
  - sobrescrita detectada: aviso explícito de replace do runtime ativo,
  - erro: mensagem objetiva, exit code não-zero.
- Integrar comando novo na montagem de comandos (`Seed4JCommandsFactory`) e no fixture de testes de CLI.

#### Validation

- `./mvnw -Dtest=Seed4JCommandsFactoryTest,ExtensionInstallCommandTest test`
- `./mvnw -Dtest=ExternalConfigTest test`

#### Acceptance Criteria

- `seed4j extension --help` lista `install`.
- `seed4j extension install ...` com insumos válidos retorna exit code 0.
- Saída de sucesso inclui sugestão de validação (`seed4j --version` e `seed4j list`).
- Fluxo em modo já `extension` continua instalando/sobrescrevendo (idempotente no contrato).

### Milestone 3 - Documentação e validação end-to-end

#### Goal

Atualizar contrato público e garantir que pipeline local continua íntegro.

#### Changes

- Atualizar `documentation/Commands.md`:
  - adicionar seção do comando `seed4j extension install`,
  - substituir/encadear fluxo manual para fluxo automatizado,
  - manter exemplos de validação pós-instalação.
- Atualizar menções relevantes em `README.md` apenas se necessário para descoberta do comando.
- Garantir que mensagens de erro/sucesso descritas em docs batem com o comportamento real.

#### Validation

- `./mvnw clean verify`
- `npm run prettier:check`

#### Acceptance Criteria

- Guia de comandos descreve o novo caminho oficial para instalação de extensão.
- Build/coverage/checkstyle/itens de qualidade passam no `clean verify`.

## Progress

- [x] Milestone 1 started
- [x] Milestone 1 completed
- [x] Milestone 2 started
- [x] Milestone 2 completed
- [ ] Milestone 3 started
- [ ] Milestone 3 completed

## Decisions

- Decision: `config.yml` inválido deve falhar e informar, sem “autocorreção”.
  Rationale: evita mascarar configuração quebrada e mantém comportamento explícito.
  Date/Author: 2026-05-25 / Renan + Codex

- Decision: instalação sobrescreve extensão ativa sem `--force`.
  Rationale: manter MVP simples e alinhado ao “slot único” de runtime ativo.
  Date/Author: 2026-05-25 / Renan + Codex

- Decision: `config.yml` ausente será criado com `seed4j.runtime.mode: extension`.
  Rationale: reduzir fricção operacional e completar fluxo em comando único.
  Date/Author: 2026-05-25 / Renan + Codex

- Decision: perda de formatação/comentários do `config.yml` é aceitável no MVP.
  Rationale: priorizar comportamento correto; preservação textual não é requisito agora.
  Date/Author: 2026-05-25 / Renan + Codex

- Decision: pós-sucesso sempre sugerir validação com `seed4j --version` e `seed4j list`.
  Rationale: verificação rápida de modo/distribution e descoberta de módulos extension.
  Date/Author: 2026-05-25 / Renan + Codex

## Risks and Mitigations

- Risk: sobrescrita silenciosa de extensão ativa por erro de caminho do usuário.
  Mitigation: emitir aviso explícito de replace quando runtime ativo já existir.

- Risk: divergência entre validação de `config.yml` no install e no bootstrap.
  Mitigation: concentrar validação em lógica compartilhada no domínio bootstrap.

- Risk: escrita parcial em falha de IO.
  Mitigation: usar gravação em arquivo temporário e move com replace.

- Risk: regressão de help/árvore de comandos.
  Mitigation: testes de integração para `--help`, parse de argumentos e exit codes.

## Validation Strategy

1. Rodar testes focados do domínio de instalação e comando.
2. Rodar testes da árvore de comandos e cenários de configuração externa.
3. Rodar validação completa do repositório (`./mvnw clean verify`).
4. Validar manualmente cenário feliz:
   - executar `seed4j extension install ...`,
   - executar `seed4j --version`,
   - executar `seed4j list`,
   - confirmar modo `extension` e distribution esperada.

## Rollout and Recovery

- Rollout: entregar em release normal da CLI; comando é aditivo e não quebra contratos existentes de `list/apply`.
- Recovery:
  - para desativar extensão rapidamente: ajustar `seed4j.runtime.mode: standard` em `~/.config/seed4j-cli/config.yml`;
  - para voltar versão anterior da extensão: reinstalar JAR/metadata anteriores com o mesmo comando;
  - se instalação falhar por config inválido: corrigir `config.yml` e repetir instalação.

## Lessons Learned

- O contrato atual já opera em “slot único ativo”; tratar MVP como automação desse contrato evita superdesign.
- A ordem das mutações importa: validar primeiro, habilitar `mode: extension` apenas no caminho de sucesso.
- Mesmo com sobrescrita sem `--force`, UX precisa transparência explícita de replace para evitar erro operacional.
- Reusar `RuntimeModeConfigReader` e `RuntimeExtensionJarLayoutValidator` no instalador elimina divergência semântica entre bootstrap e instalação.
- Escrita com arquivo temporário + `move` com `REPLACE_EXISTING` (e fallback sem `ATOMIC_MOVE`) mantém atualização de runtime/config defensiva em diferentes file systems.
- O `CliFixture` precisa registrar novos comandos raiz para evitar divergência entre árvore de comandos testada e árvore montada por componentes no runtime real.
