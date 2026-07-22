# Criar e integrar a skill refactor-design

Este ExecPlan é um documento vivo. Manter `Progress`, `Decisions`, `Risks and Mitigations` e `Lessons Learned` atualizados durante a execução.

## Purpose / Big Picture

Criar a skill pessoal `refactor-design` para revisar sistematicamente o desenho post-green, aplicar refatorações que preservem comportamento e evitar que heurísticas evolutivas de qualidade sejam acumuladas no `AGENTS.md`. Integrar a revisão ao workflow `seed4j-execplan-tdd`, preservando TDD como responsável por descobrir comportamento e deixando a consolidação estrutural antes da validação final.

## Scope

Em escopo: criar `/home/renanfranca/.codex/skills/refactor-design` com `SKILL.md`, metadata de UI e duas referências; integrar `seed4j-execplan-tdd`; reconciliar sua edição local existente; enxugar o `AGENTS.md`; validar estrutura, formatação, descoberta e comportamento da skill; e definir evolução controlada sem automodificação ordinária.

Fora de escopo: modificar `tdd-behavior-autonomous-quiet`; alterar `apply-set` ou produção Java; refatorar os achados atuais; criar scripts, assets, plugin, commit ou push; e editar caches ou cópias instaladas fora do repositório-fonte.

## Definitions

- **Post-green design review:** revisão estrutural depois que o comportamento solicitado e seu checkpoint público estão verdes.
- **Behavior-preserving refactor:** mudança estrutural sem alteração de contrato público, saída observável, estado persistido ou regra de negócio.
- **Design finding:** evidência de dependência, estado, responsabilidade ou representação inadequada.
- **Heuristic candidate:** aprendizado potencialmente reutilizável ainda não promovido a regra.
- **Controlled skill evolution:** atualização explícita, revisável e versionada da fonte da skill; não é autoaprendizado.
- **Forward test:** uso da skill em contexto novo e descartável para avaliar generalização.

## Existing Context

`/home/renanfranca/.codex/skills` é o repositório Git `renanfranca/codex-skills` e também a fonte pessoal descoberta pelo Codex. Antes desta execução ele contém somente uma alteração local em `seed4j-execplan-tdd/SKILL.md`: a tentativa de corrigir o diretório compartilhado deixou um backtick antes de `/shared`. Essa intenção deve ser preservada e reconciliada sem reset. O mesmo arquivo usa `tdd-behavior-autonomous-quiet` no workflow, mas ainda menciona `tdd-strict-autonomous-quiet` na descrição. No Seed4J, `AGENTS.md` mistura invariantes permanentes com heurísticas detalhadas de revisão.

Estado inicial observado em 2026-07-22:

- Seed4J CLI: `?? _temporary/`; `AGENTS.md` sem diff.
- Repositório de skills: `M seed4j-execplan-tdd/SKILL.md` com a única alteração do caminho mal formatado.
- `/home/renanfranca/.codex/skills/refactor-design` ainda não existe.

## Desired End State

`refactor-design` deve conter apenas `SKILL.md`, `agents/openai.yaml`, `references/design-review-rubric.md` e `references/java-spring-hexagonal.md`. A skill deve exigir entry gate verde, revisão limitada à mudança, classificação e evidência antes de agir, refatoração protegida por testes comportamentais, retorno ao TDD diante de comportamento ausente, output quiet e evolução controlada sem automodificação. `seed4j-execplan-tdd` deve compor ExecPlan, TDD comportamental, revisão post-green e validação final nessa ordem. `AGENTS.md` deve conservar invariantes e apenas rotear brevemente a revisão.

## Milestones

### Milestone 1 — Inicializar a skill

#### Goal

Criar a estrutura oficial mínima pelo `init_skill.py` em `/home/renanfranca/.codex/skills/refactor-design`.

#### Changes

Executar o inicializador com `--resources references` e metadata `Refactor Design`, sem scripts, assets ou documentação auxiliar.

#### Validation

Executar `quick_validate.py /home/renanfranca/.codex/skills/refactor-design` e confirmar que não restam placeholders.

#### Acceptance Criteria

Nome exato, arquivos mínimos presentes e frontmatter válido.

### Milestone 2 — Definir o workflow post-green

#### Goal

Escrever `SKILL.md` conciso e imperativo.

#### Changes

Definir trigger post-green, entry gate, escopo, roteamento das referências, classificação, exigência de evidência, loop de refatoração, gates, output quiet e consolidação de aprendizados sem automodificação.

#### Validation

Ler o arquivo completo, confirmar menos de 500 linhas e executar `quick_validate.py`.

#### Acceptance Criteria

TDD e consolidação ficam distintos; comportamento ausente retorna ao TDD; nenhuma automodificação é autorizada.

### Milestone 3 — Criar a rubrica geral

#### Goal

Criar `references/design-review-rubric.md` como catálogo investigativo e não mecânico.

#### Changes

Cobrir acoplamento temporal, estado escondido ou longevo, builders com efeitos colaterais, snapshots inconsistentes, vazamento de apresentação/interface, transformações e mapeamentos frágeis, valores vazios como status, políticas independentes, conceitos de posição, metadata de framework, abstrações de teste e conceitos escondidos por tipos genéricos/primitivos. Cada item deve trazer sinal, risco, perguntas, opções, falsos positivos e condição de não ação.

#### Validation

Confirmar neutralidade tecnológica, ausência de regras de Seed4J/Picocli e índice quando exceder 100 linhas.

#### Acceptance Criteria

Aplicável fora de Java e sem impor padrões por checklist.

### Milestone 4 — Criar a referência Java/Spring/hexagonal

#### Goal

Criar `references/java-spring-hexagonal.md` com variantes técnicas condicionais.

#### Changes

Cobrir estado em singleton Spring, lifecycle, constructor injection, records defensivos, `Optional.get()`, `Object` como negócio, enums entre contextos, adapters, ports versus seams, composition pré-Spring, vazamento técnico e enforcement automatizável.

#### Validation

Confirmar carregamento condicional, compatibilidade com `AGENTS.md`, exemplos genéricos e índice quando necessário.

#### Acceptance Criteria

Complementar a rubrica geral sem afetar projetos não Java.

### Milestone 5 — Integrar seed4j-execplan-tdd

#### Goal

Adicionar a revisão post-green antes da validação final.

#### Changes

Corrigir descrição e caminho em `seed4j-execplan-tdd/SKILL.md`, preservar a intenção da edição local, adicionar `refactor-design`, manter o ExecPlan vivo e regenerar `agents/openai.yaml`.

#### Validation

Executar `quick_validate.py` e `rg -n "tdd-strict-autonomous-quiet|refactor-design|seed4j-cli-ai-context" /home/renanfranca/.codex/skills/seed4j-execplan-tdd`.

#### Acceptance Criteria

Sem referência obsoleta; nova skill explicitamente composta entre TDD verde e validação final.

### Milestone 6 — Enxugar e rotear AGENTS.md

#### Goal

Manter invariantes permanentes no repositório e mover a rubrica evolutiva para a skill.

#### Changes

Remover snapshot de metadata, acoplamento temporal, estado de execução em singleton, idempotência de builders, transformação compartilhada, vazio como status, divisão de services e índice de posições. Preservar boundaries, TDD, domínio estruturado, enums explícitos e validação. Adicionar uma instrução curta post-green em `Agent Validation Behavior` usando `refactor-design` quando disponível.

#### Validation

Executar `npm run prettier:check`, `git diff --check` e inspecionar `git diff -- AGENTS.md`. Não executar Maven porque não há produção Java alterada.

#### Acceptance Criteria

Rubrica não duplicada e invariantes preservados.

### Milestone 7 — Validar descoberta e comportamento

#### Goal

Validar as duas skills e forward-testar comportamento sem contaminar repositórios reais.

#### Changes

Executar validações estruturais e de Git. Para forward tests, usar diretório descartável e contexto novo nos cenários finding, no-action, behavior gate e evolution policy. Solicitar autorização antes de iniciar outro agente ou sessão caso a execução envolva esse recurso.

#### Validation

Executar `quick_validate.py` nas duas skills, `git diff --check` e `git status --short` nos dois repositórios. Inspecionar artefatos dos forward tests e repetir a validação após ajustes.

#### Acceptance Criteria

A skill é descoberta, age por evidência, aceita `No action`, retorna ao TDD quando necessário, não se automodifica e não altera os repositórios reais nos testes.

## Progress

- [x] ExecPlan salvo.
- [x] Estado inicial dos dois repositórios registrado.
- [x] Milestone 1 iniciado.
- [x] Milestone 1 concluído.
- [x] Milestone 2 iniciado.
- [x] Milestone 2 concluído.
- [x] Milestone 3 iniciado.
- [x] Milestone 3 concluído.
- [x] Milestone 4 iniciado.
- [x] Milestone 4 concluído.
- [x] Milestone 5 iniciado.
- [x] Milestone 5 concluído.
- [x] Milestone 6 iniciado.
- [x] Milestone 6 concluído.
- [x] Milestone 7 iniciado.
- [x] Milestone 7 concluído.

## Decisions

- Decision: nomear a skill `refactor-design`.
  Rationale: nome curto, verb-led e específico para revisão estrutural.
  Date/Author: 2026-07-22 / Renan França e Codex
- Decision: usar `/home/renanfranca/.codex/skills` como fonte canônica.
  Rationale: o diretório já é repositório Git e fonte descoberta pelo ambiente.
  Date/Author: 2026-07-22 / Renan França e Codex
- Decision: manter uma referência geral e outra condicional para Java/Spring/hexagonal.
  Rationale: progressive disclosure sem fragmentação prematura.
  Date/Author: 2026-07-22 / Renan França e Codex
- Decision: deixar no `AGENTS.md` somente invariantes e routing post-green.
  Rationale: heurísticas evolutivas não devem ocupar o contexto de toda tarefa.
  Date/Author: 2026-07-22 / Renan França e Codex
- Decision: proibir automodificação em revisões normais.
  Rationale: evolução deve ser explícita, revisável, versionada e forward-tested.
  Date/Author: 2026-07-22 / Renan França e Codex

## Risks and Mitigations

- Risk: sobrescrever a alteração local em `seed4j-execplan-tdd/SKILL.md`.
  Mitigation: registrar o diff inicial, editar por patch localizado e revisar o diff final sem reset.
- Risk: modificar comportamento sob o nome de refatoração.
  Mitigation: entry gate, behavior gate, testes públicos e retorno obrigatório ao TDD.
- Risk: refatorar mecanicamente por checklist.
  Mitigation: exigir evidência, risco, custo, falsos positivos e permitir `No action`.
- Risk: duplicar conteúdo entre skill, referências e `AGENTS.md`.
  Mitigation: workflow no `SKILL.md`, heurísticas nas referências e invariantes no repositório.
- Risk: skill pessoal indisponível a contribuidores.
  Mitigation: usar “when available” no `AGENTS.md` e composição explícita no workflow pessoal.
- Risk: crescimento ilimitado ou mudança das regras durante a execução.
  Mitigation: evolução apenas em tarefa explícita, sem registrar achados contextuais nem automodificar.
- Risk: forward tests contaminarem repositórios.
  Mitigation: diretórios temporários, contexto mínimo e autorização antes de nova sessão/agente. Os quatro testes foram executados somente em `/tmp`; hashes confirmaram que a skill não foi automodificada.
- Risk: alterações serem commitadas sem solicitação explícita durante a execução.
  Mitigation: nenhum comando de commit foi executado por esta sessão ou pelos forward tests. Foram detectados commits externos `d660080` no repositório de skills e `bc3fbd9` no Seed4J; não os reverter nem reescrever automaticamente e relatar o fato ao usuário.

## Validation Strategy

1. Validar estrutura e frontmatter com `quick_validate.py`.
2. Validar metadata de UI regenerada e roteamento das referências.
3. Validar Markdown do Seed4J com Prettier.
4. Validar whitespace e escopo com `git diff --check`, `git diff` e `git status` nos dois repositórios.
5. Forward-testar finding, no-action, behavior gate e evolution policy em ambientes descartáveis.
6. Confirmar descoberta em contexto novo ou após refresh.
7. Não executar Maven, pois nenhuma produção Java será alterada.

## Rollout and Recovery

A skill será descoberta diretamente do repositório pessoal. Se não aparecer, iniciar nova sessão ou reiniciar Codex. Não criar commits automaticamente; manter os diffs dos dois repositórios separados. Diante de refatoração excessiva, remover temporariamente a composição somente após confirmação, ajustar gates ou rubrica e repetir forward tests. Não usar reset, checkout destrutivo ou remoção automática de diretórios.

## Lessons Learned

- `/home/renanfranca/.codex/skills` é simultaneamente repositório-fonte e diretório pessoal descoberto.
- `seed4j-execplan-tdd/SKILL.md` possui uma alteração local que deve ser integrada.
- A skill complementa o refactor local do TDD; não o substitui.
- Evolução iterativa é suportada, mas automodificação durante uma revisão comum não faz parte do modelo.
- O inicializador oficial criou exatamente a estrutura solicitada; a substituição do template não deixou placeholders e `quick_validate.py` aceitou a primeira versão.
- `npm run prettier:check` e `git diff --check` passaram após o enxugamento do `AGENTS.md`; nenhuma validação Maven é necessária porque não houve mudança em produção Java.
- As duas skills passaram em `quick_validate.py`; `git diff --check` passou nos dois repositórios e a nova skill contém somente os quatro arquivos planejados.
- A autorização explícita foi recebida e os forward tests do Milestone 7 foram concluídos em diretórios descartáveis.
- A nova etapa confirmou que `refactor-design` é descoberta pelo ambiente como skill disponível.
- O finding test classificou estado de invocação escondido como `Design risk`, tornou os dados locais e manteve os dois testes verdes sem alterar testes.
- O no-action test classificou uma função pequena e coesa como `No action` e não criou abstrações.
- O behavior-gate test encontrou uma suíte vermelha, classificou o comportamento incompleto como `Defect` e encerrou sem editar fonte.
- O evolution-policy test reconheceu que o aprendizado já estava coberto, não registrou regra nova e não alterou a skill; os hashes de `SKILL.md` e das referências permaneceram idênticos.
- Durante a validação final apareceram commits externos `d660080` (`feat(refactor-design): add post-green design review skill`) e `bc3fbd9` (`docs(agents): route post-green design reviews`). Esta sessão não executou commits; o estado foi preservado sem reset ou reescrita.
- A validação final passou: as duas skills foram aceitas por `quick_validate.py`, `npm run prettier:check` passou, `git diff --check` passou nos dois repositórios e os quatro diretórios temporários foram removidos.
