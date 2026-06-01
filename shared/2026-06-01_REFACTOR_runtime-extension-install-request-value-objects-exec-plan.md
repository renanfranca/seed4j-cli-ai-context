# Tipar RuntimeExtensionInstallRequest com Value Objects

This ExecPlan is a living document. Keep Progress, Decisions, Risks, and Lessons Learned up to date as work advances.

## Purpose / Big Picture

Refatorar a instalação de runtime extension para que os parâmetros de RuntimeExtensionInstallRequest representem conceitos de domínio, não primitivos soltos. O comportamento externo deve continuar igual:
seed4j extension install <jar> --distribution-id ... --distribution-version ... instala o jar, persiste metadata e mostra a mesma saída.

## Scope

In-scope: criar value objects para path textual do jar, distribution id e distribution version; propagar id/version para RuntimeMetadata e RuntimeSelection; adaptar consumidores e testes.

Out-of-scope: mudar formato do YAML, mudar opções CLI, validar formato semântico de id/version além de não-blank, ou trocar RuntimeSelection.extensionJarPath() de Optional<Path> para VO.

Safety boundary: This task is limited to authorized, defensive maintenance of this repository. Do not provide offensive guidance or policy-bypassing instructions.

## Definitions

RuntimeExtensionJarPath: value object público que armazena java.nio.file.Path e expõe factory from(String) para transformar a entrada textual da borda CLI em tipo de domínio.

RuntimeDistributionId: value object público para o identificador da distribuição da runtime extension.

RuntimeDistributionVersion: value object público para a versão da distribuição da runtime extension.

RuntimeSelection: modelo público que descreve qual runtime está ativa quando a CLI roda.

## Existing Context

RuntimeExtensionInstallRequest agora recebe RuntimeExtensionJarPath, RuntimeDistributionId e RuntimeDistributionVersion.

ExtensionInstallCommand recebe strings do picocli e constrói RuntimeExtensionJarPath via RuntimeExtensionJarPath.from(...).

RuntimeExtensionInstaller valida o layout do jar chamando RuntimeExtensionJarLayoutValidator.validate(request.extensionJarPath().path()).

FileSystemRuntimeExtensionArtifactsRepository copia o jar a partir de request.extensionJarPath().path() e escreve metadata.yml com distribution.id e distribution.version.

RuntimeMetadata lê o YAML como strings e RuntimeSelection expõe Optional<String> para id/version, usados por Seed4JCliLauncher, SystemPropertyRuntimeSelectionProvider e Seed4JCommandsFactory.

## Desired End State

RuntimeExtensionInstallRequest deve ficar assim conceitualmente:

public record RuntimeExtensionInstallRequest(
RuntimeExtensionJarPath extensionJarPath,
RuntimeDistributionId distributionId,
RuntimeDistributionVersion distributionVersion
) {}

RuntimeExtensionJarPath deve armazenar Path path, validar Assert.notNull("path", path) e Assert.notBlank("path", path.toString()) no construtor canônico, expor factory RuntimeExtensionJarPath.from(String) com Assert.notBlank("path", path), e não ter método get().

RuntimeDistributionId deve armazenar String id, validar Assert.notBlank("id", id), e não ter método get().

RuntimeDistributionVersion deve armazenar String version, validar Assert.notBlank("version", version), e não ter método get().

RuntimeMetadata e RuntimeSelection devem usar RuntimeDistributionId e RuntimeDistributionVersion; conversão para String deve acontecer somente na borda, usando .id() e .version().

## Milestones

### Milestone 1 - Registrar ExecPlan

#### Goal

Criar o documento vivo antes da implementação.

#### Changes

- [x] Criar shared/2026-06-01_REFACTOR_runtime-extension-install-request-value-objects-exec-plan.md com este plano.
- [x] Manter Progress, Decisions, Risks and Mitigations e Lessons Learned atualizados durante a execução.

#### Validation

- [x] Command: git status --short
- [x] Expected result: o novo arquivo de plano aparece como mudança não commitada.

#### Acceptance Criteria

- [ ] O plano é autocontido e contém todos os milestones necessários para outro agente implementar.
- [x] O plano é autocontido e contém todos os milestones necessários para outro agente implementar.

### Milestone 2 - Introduzir Value Objects

#### Goal

Adicionar os tipos de domínio públicos sem alterar ainda o comportamento externo.

#### Changes

- [ ] Criar src/main/java/com/seed4j/cli/bootstrap/domain/RuntimeExtensionJarPath.java.
- [ ] Criar src/main/java/com/seed4j/cli/bootstrap/domain/RuntimeDistributionId.java.
- [ ] Criar src/main/java/com/seed4j/cli/bootstrap/domain/RuntimeDistributionVersion.java.
- [ ] Usar Assert.notBlank nos três VOs.
- [ ] Em RuntimeExtensionJarPath, implementar from(String) retornando RuntimeExtensionJarPath com Path.of(path).
- [ ] Não criar método get() em nenhum dos novos VOs.

#### Validation

- [ ] Command: ./mvnw -Dtest=RuntimeExtensionInstallRequestTest test
- [ ] Expected result: os testes novos dos VOs passam.

#### Acceptance Criteria

- [ ] RuntimeExtensionJarPath.from(input).path() retorna o mesmo Path de Path.of(input).
- [ ] path, id e version blank/null falham por validação de domínio.

### Milestone 3 - Tipar Request, Metadata e Selection

#### Goal

Propagar os VOs pelo domínio de bootstrap sem mudar a experiência da CLI.

#### Changes

- [ ] Atualizar RuntimeExtensionInstallRequest para usar os três VOs.
- [ ] Atualizar RuntimeExtensionInstaller para validar request.extensionJarPath().path().
- [ ] Atualizar FileSystemRuntimeExtensionArtifactsRepository para copiar de request.extensionJarPath().path() e escrever YAML com request.distributionId().id() e request.distributionVersion().version().
- [ ] Atualizar RuntimeMetadata para armazenar RuntimeDistributionId e RuntimeDistributionVersion.
- [ ] Atualizar RuntimeSelection para expor Optional<RuntimeDistributionId> e Optional<RuntimeDistributionVersion>, mantendo Optional<Path> extensionJarPath.
- [ ] Atualizar Seed4JCliLauncher, SystemPropertyRuntimeSelectionProvider e Seed4JCommandsFactory para converter VOs para string somente ao escrever system properties ou renderizar versão.

#### Validation

- [ ] Command: ./mvnw
      -Dtest=RuntimeExtensionInstallerTest,RuntimeExtensionApplicationServiceTest,RuntimeExtensionApplicationConfigurationTest,RuntimeSelectionTest,Seed4JCliLauncherTest,SystemPropertyRuntimeSelectionProviderTest,
      CurrentProcessRuntimeSelectionProviderTest,Seed4JCommandsFactoryTest test

- [ ] Expected result: todos os testes focados compilam e passam.

#### Acceptance Criteria

- [ ] O YAML gerado continua contendo distribution.id e distribution.version com os mesmos valores.
- [ ] seed4j --version continua mostrando Distribution ID e Distribution version iguais.
- [ ] O child process continua recebendo as mesmas system properties textuais.

### Milestone 4 - Adaptar Borda CLI e Testes

#### Goal

Fazer a borda CLI construir os VOs explicitamente e limpar os testes afetados.

#### Changes

- [ ] Atualizar ExtensionInstallCommand para construir RuntimeExtensionJarPath.from(extensionJarPath), new RuntimeDistributionId(distributionId) e new RuntimeDistributionVersion(distributionVersion).
- [ ] Atualizar testes que constroem RuntimeExtensionInstallRequest diretamente para usar VOs.
- [ ] Atualizar testes de RuntimeSelection para comparar .id() e .version() ou comparar os VOs explicitamente.
- [ ] Usar helpers de teste apenas quando houver reutilização clara; evitar helpers de uma linha com um único call site.

#### Validation

- [ ] Command: ./mvnw -Dtest=ExtensionInstallCommandTest,RuntimeSelectionTest,RuntimeExtensionInstallerTest test
- [ ] Expected result: instalação via comando, seleção de runtime e persistência continuam passando.

#### Acceptance Criteria

- [ ] A CLI continua aceitando os mesmos argumentos.
- [ ] Os testes deixam claro onde há conversão de string de entrada para domínio tipado.

### Milestone 5 - Validação Completa

#### Goal

Confirmar que a refatoração não introduziu regressões de compilação, formatação ou comportamento.

#### Changes

- [ ] Rodar formatter se necessário durante implementação.
- [ ] Atualizar o ExecPlan com progresso final, decisões e aprendizados.

#### Validation

- [ ] Command: npm run prettier:check
- [ ] Expected result: formatação válida.
- [ ] Command: ./mvnw clean verify
- [ ] Expected result: build completo, testes, Checkstyle e cobertura passam.

#### Acceptance Criteria

- [ ] Nenhuma alteração de UX ou formato de metadata além da tipagem interna.
- [ ] Não existem usos restantes de new RuntimeExtensionInstallRequest(Path, String, String).
- [ ] Não existem métodos get() nos novos VOs.

## Progress

- [x] Milestone 1 started
- [x] Milestone 1 completed
- [x] Milestone 2 started
- [x] Milestone 2 completed
- [x] Milestone 3 started
- [x] Milestone 3 completed
- [x] Milestone 4 started
- [x] Milestone 4 completed
- [ ] Milestone 5 started
- [ ] Milestone 5 completed

## Decisions

- Decision: RuntimeExtensionJarPath armazena Path e expõe RuntimeExtensionJarPath.from(String).
  Rationale: o request tipado precisa carregar um Path válido internamente, mas a borda CLI continua textual e deve converter por factory explícita com validação.
  Date/Author: 2026-06-01 / Codex + Renan

- Decision: não criar método get() nos novos VOs.
  Rationale: records já expõem accessors nomeados (path(), id(), version()), e o método extra não agrega semântica.
  Date/Author: 2026-06-01 / Renan

- Decision: manter RuntimeSelection.extensionJarPath() como Optional<Path>.
  Rationale: esse valor vem da configuração ativa e é consumido por validadores/loaders de filesystem; trocar para VO textual adicionaria conversões sem benefício claro.
  Date/Author: 2026-06-01 / Codex

- Decision: propagar RuntimeDistributionId e RuntimeDistributionVersion para RuntimeMetadata e RuntimeSelection.
  Rationale: evita voltar para primitivos logo após ler metadata e mantém os conceitos de distribuição tipados no domínio.
  Date/Author: 2026-06-01 / Codex + Renan

- Decision: manter o construtor canônico público do record e reforçar uso de from(String) via call sites.
  Rationale: Java mantém o construtor canônico de record público; a regra "não usar new direto" será aplicada por busca e revisão nos consumidores.
  Date/Author: 2026-06-01 / Codex + Renan

## Risks and Mitigations

- Risk: tipos públicos novos podem quebrar construtores usados fora do pacote.
  Mitigation: criar os VOs como public record em arquivos próprios e adaptar todos os call sites.

- Risk: Optional<String> em RuntimeSelection virar Optional<VO> pode quebrar renderização e system properties.
  Mitigation: converter explicitamente com .id() e .version() somente nas bordas.

- Risk: validação not-blank em system properties pode expor propriedades internas inválidas que antes passavam silenciosamente.
  Mitigation: tratar isso como endurecimento aceitável porque essas properties são geradas pela própria CLI; testes devem cobrir o caminho normal.

## Validation Strategy

1. Criar testes focados para os novos VOs.
2. Rodar testes dos pacotes de bootstrap e primary command afetados.
3. Rodar npm run prettier:check.
4. Rodar ./mvnw clean verify.
5. Conferir manualmente no diff que o YAML e a saída de versão continuam textualmente iguais.

## Rollout and Recovery

A mudança é refatoração interna sem migração de arquivo de configuração. Se houver regressão, reverter o commit restaura o contrato antigo Path/String/String. Como o YAML persistido não muda, nenhum usuário precisa limpar ~/.config/seed4j-cli.

## Lessons Learned

Seed4JProjectFolder em seed4j/module é a referência mais próxima para armazenar string e expor Path por método de domínio.

RuntimeExtensionInstallRequest é público e usado fora de bootstrap.domain; por isso seus novos tipos também precisam ser públicos.

Como RuntimeExtensionJarPath é record público, o construtor canônico continua público; o alinhamento com o factory depende de disciplina em call sites e testes.

RuntimeSelection tem dois papéis diferentes: jar path operacional de filesystem e metadata de distribuição. Só a metadata precisa ser propagada como VO nesta refatoração.
