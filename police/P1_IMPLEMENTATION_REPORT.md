---
title: MZ Framework — Relatório de implementação P1 do sistema policial
status: TEST_REPORT
authority: LEVEL_4_REFERENCE
owner: MZ Police
last_reviewed: 2026-10-14
applies_to: [mz_police, mz_core, mz_org, mz_starter]
related_contracts: [MZ-IDENTITY-v2, MZ-DOCS-v1]
---

# Relatório de implementação P1

## 1. Resumo

A Wave P1 criou a fundação server-authoritative de `mz_police`. O resource contém somente configuração de servidor, bridge para contratos públicos canônicos, auditoria, policy centralizada e inicialização. Não possui NUI, client script, evento de rede, export público, persistência ou gameplay.

Estado: implementação estática concluída; validação dentro do runtime FiveM permanece pendente.

## 2. Arquivos criados e alterados

### Novo resource `mz_police`

- `fxmanifest.lua`
- `.gitignore`
- `config/server.lua`
- `server/bridge.lua`
- `server/audit.lua`
- `server/policy.lua`
- `server/main.lua`
- `server/p1_gate.lua`: removido na limpeza estrutural; era harness temporário de staging
- `tests/p1_harness.lua`: removido da distribuição; preservado apenas em QA externo/Git

### Integração e documentação

- `mz_starter/cfg/resources.cfg`: inicializa `mz_police` após `mz_org`.
- `mz_starter/tools/install_deps.bat` e `install_deps.sh`: mantêm a instalação limpa na mesma ordem do boot manifest e resolvem `mz_police` pela distribuição local.
- `mz_starter/tools/check_structure.bat`: exige o manifesto do novo resource.
- `mz_docs/police/P0_AUDIT_AND_PLAN.md`: esclarece que o `dutyPoint` bloqueia validação operacional, não a fundação P1.
- `mz_docs/police/P1_IMPLEMENTATION_REPORT.md`: este relatório.
- `mz_docs/police/P1_STAGING_VALIDATION_GATE.md`: roteiro, matriz e evidências do gate runtime.
- `mz_docs/INDEX.md`: entrada do relatório P1.

## 3. Contratos reais utilizados

| Owner | Contrato | Uso |
|---|---|---|
| `mz_core` | `ResolvePlayerIdentity(source)` | Resolve sessão, account, personagem ativo e `citizenid`; a policy exige `active == true` e campos consistentes |
| `mz_core` | `CreateDetailedLog(scope, action, payload)` | Registra inicialização e rejeições relevantes |
| `mz_org` | `GetOrganizationReadiness()` | Impede autorização enquanto o owner não estiver pronto |
| `mz_org` | `GetPlayerOrgContext(source)` | Obtém membership, grade e duty autoritativos |
| `mz_org` | `CanOrg(source, orgCode, capability)` | Decide a capability no owner canônico |

As chamadas usam o proxy FiveM em formato method-style e são protegidas por estado do resource, existência do export e `pcall`.

## 4. Capabilities

Nenhuma capability foi adicionada. A P1 não possui ação operacional que justifique reservar capabilities das waves futuras.

O harness usa `org.view`, já existente e concedida no seed policial, apenas para verificar o contrato da policy. Qualquer capability `staff.*` é rejeitada antes de consultar `CanOrg`, preservando a separação entre autoridade staff global e polícia.

## 5. Decisões de segurança

- `Policy.authorize(source, capability, context)` é interna; não há export, callback ou evento público.
- Somente `police` está habilitada em `AuthorizedOrganizations`.
- Identidade, sessão, organização, grade, duty e capability são resolvidos pelos owners no servidor.
- Toda autorização P1 exige duty ativo; não existe flag de bypass.
- O contexto aceita apenas `orgCode`; campos como `citizenid`, `duty` ou permissions causam `invalid_context`.
- Capabilities são normalizadas, limitadas e `staff.*` é recusada.
- Falhas de dependência, export, sessão, readiness ou dados inconsistentes negam a autorização.
- Resultados têm `{ ok, code }`; sucesso inclui snapshots mínimos de identidade e organização.
- Rejeições resolvidas são auditadas com cooldown por source, sessão, decisão, capability e organização.
- O throttle é limpo em `playerDropped`; uma nova sessão possui chave própria.
- A auditoria não registra identifiers externos, tokens, inventário, dinheiro ou payload client-side.
- Nenhuma falha de autorização aciona ban ou `mz_banguard`.

## 6. Testes executados

Executados em 2026-10-13, localmente, fora do runtime FiveM:

| Validação | Resultado | Evidência |
|---|---|---|
| Parse de todos os arquivos Lua do resource com `luac -p` | PASS | Sem erro de sintaxe |
| `lua tests/p1_harness.lua` | PASS | `mz_police P1 harness: PASS cases=23` |
| Manifesto e ausência de client/persistência | PASS | Assertions do harness |
| Sessão/personagem válidos e inválidos | PASS | Assertions do harness |
| Organização policial e não policial | PASS | Assertions do harness |
| Capability permitida, negada e `staff.*` | PASS | Assertions do harness |
| Duty ativo/inativo | PASS | Assertions do harness |
| Contexto falsificado | PASS | Assertions do harness |
| Core/Org ausentes e Org não pronta | PASS | Assertions do harness |
| Throttle de auditoria e inicialização | PASS | Assertions do harness |
| `node --test tests/clean_install_manifest.test.js` em `mz_starter` | PASS | 4 testes de ordem, preflight e dependências |
| Todos os testes Node de `mz_starter` | PASS | 9 testes |
| `bash -n tools/install_deps.sh tools/check_structure.sh` via Git Bash | PASS | Sem erro de sintaxe shell |
| Metadados e links internos da documentação policial | PASS | Validação local |
| `git diff --check` nos arquivos tracked afetados e whitespace dos arquivos novos | PASS | Sem erro; apenas avisos de conversão LF/CRLF |

Os testes usam mocks explícitos para os proxies de exports e eventos FiveM. Eles comprovam a lógica local e os contratos esperados, não a integração real no servidor.

## 7. Lacunas e bloqueios

- O seed da organização `police` continua sem `dutyPoint`; nenhuma coordenada foi inventada.
- Não existe nesta wave ação para entrada/saída de duty. O fluxo continua pertencendo a `mz_org`.
- Não foi executado servidor FiveM com sessão, membership e banco reais.
- P2 ainda depende do desenho de cleanup de custódia e, posteriormente, de grants seguros em `mz_inventory`.

## 8. Riscos restantes

- Divergência entre mocks e comportamento do runtime FiveM deve ser descartada em staging.
- O resource inicia mesmo quando `mz_org` ainda está preparando seu seed, mas todas as autorizações continuam fail-closed até `GetOrganizationReadiness().ready == true`.
- Sem `dutyPoint`, não é possível validar o ciclo operacional completo de serviço.

## 9. Preparação para P2

Antes de iniciar P2:

1. Configurar e validar o `dutyPoint` no owner `mz_org`.
2. Subir `mz_police` em staging e testar identidade, readiness, duty e auditoria com jogadores reais.
3. Aprovar a state machine de custódia runtime e seus cleanup triggers.
4. Definir o contrato mínimo de grant cross-player em `mz_inventory` antes de implementar busca/apreensão.
5. Manter P2 sem persistência de custódia, sem `source` como identidade e sem evento interno do Core tratado como API.

## Impacto arquitetural

- Contratos públicos: nenhum novo.
- Ownership canônico: inalterado.
- Writer/authority: `mz_org` continua autoridade de membership/duty/capability; `mz_core` continua autoridade de identidade/sessão/log.
- Dependências cross-resource: `mz_police` depende de `mz_core` e `mz_org`.
- Persistência/database: nenhuma.
- Segurança/permissões: nova policy interna fail-closed; nenhuma capability criada.
- Novos exports/eventos/callbacks/adapters públicos: nenhum.
- Dependências externas: nenhuma.
- Graphify: consulta inicial no grafo existente e atualização code-only restrita a `mz_police` (43 nodes, 58 edges); nenhuma reindexação global.

