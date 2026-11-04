# MZ Police — estabilização e consolidação P1–P7

Data: 2026-10-15
Referência obrigatória: `mz_docs/police/P1_P7_CONTINUITY_AUDIT.md`
Escopo: P1–P7, contratos owners, testes integrados, Git e distribuição para staging
Resultado: **gate estático e integrado aprovado; runtime FiveM/MySQL não executado e não aprovado**

## Resultado executivo

- As oito falhas do runner global foram reproduzidas, classificadas e corrigidas nas fixtures
  centralizadas. O resultado passou de **43/51** para **51/51 PASS**.
- A divergência P6→P7 apontada pela auditoria foi fechada: a sentença agora parte do `public_id` da
  acusação P6, usa seu snapshot legal persistido e é recusada quando código, nome, duração ou versão
  divergem da configuração aprovada do `mz_prison`.
- `mz_prison_sentences` passa a persistir `charge_public_id`. Schema legado sem essa coluna falha
  fechado com `schema_outdated_missing_charge_public_id`; nenhuma migração foi executada.
- A distribuição local já inclui `mz_police` e `mz_prison` na ordem correta. A documentação e o teste
  do starter foram corrigidos para enumerar ambos como `distribution-only`.
- Nenhum código P8/P9, gameplay novo, pena, multa, coordenada ou regra jurídica foi criado.

## As oito falhas integradas

Todas as oito fixtures são anteriores às waves policiais de 2026-10-14. Seis possuem baseline
versionado PASS no registry; as duas marcadas como teste novo não têm baseline confiável e permanecem
classificadas como **baseline indeterminado**. Nenhuma correção alterou lógica de produção para obter PASS.

| Teste | Baseline | Causa comprovada | Classificação e correção |
|---|---|---|---|
| `inventory_player_transfer_harness` | PASS histórico; origem `383d35b` | Fixture carregava o service extraído sem `MZEquipment`. | Fixture obsoleta; passou a carregar `mz_inventory/shared/equipment.lua`. |
| `phase_0_5_debug_surface_harness` | PASS histórico; origem `bfb14ce` | Caminhos ainda apontavam para veículos removidos do Core e exigiam um `debug.lua` legado. | Fixture obsoleta; passou a verificar o owner `mz_vehicles` e os ACEs atuais. |
| `phase_0_5_position_security_harness` | PASS histórico; origem `93c313f` | Fixture não fornecia `MZIdentityService.closeSession`, hoje consumido pelo lifecycle real. | Dependência de fixture ausente; mock mínimo do contrato real. |
| `player_state_consumer_hardening_harness` | PASS histórico; origem `d4c3107` | Assert literal não reconhecia o wrapper fail-closed existente em clothing. | Assert obsoleto; agora verifica wrapper e chamada efetivos, sem enfraquecer o guard. |
| `weapon_authority_contract_harness` | PASS histórico; origem `c219ea4` | Além de `MZEquipment` ausente, simulava sacar arma direto do inventário principal. | Fixture incompatível; agora usa slot `equipment` e `useEquipmentWeaponSlot`. |
| `weapon_inventory_reload_harness` | PASS histórico; origem `b8b4cd3` | Mesma incompatibilidade do fluxo de equipamento e roteamento incorreto de metadata entre inventários. | Fixture incompatível; separa MAIN/EQUIPMENT e valida persistência no slot equipado. |
| `player_state_org_account_guard_harness` | baseline indeterminado; teste novo | Caminho relativo ainda apontava para `mz_core`, embora o owner seja `mz_economy`. | Fixture obsoleta; caminho corrigido para o owner canônico. |
| `inventory_item_access_harness` | baseline indeterminado; teste novo | Fixture carregava o service atual sem `MZEquipment`. | Fixture obsoleta; dependência canônica carregada. |

## Contrato P6→P7 consolidado

O cliente informa apenas `targetSource`, `caseId`, `chargeId` e `requestId`. `mz_police` revalida
policy, capability, duty, custódia, distância, routing bucket, organização, caso e vínculo da acusação
ao cidadão. Código, nome, duração e versão legal são lidos da linha de `mz_police_case_charges`; não são
aceitos do cliente.

O export server-side `RequestSentence`, allowlisted exclusivamente para `mz_police`, recebe o snapshot
derivado. `mz_prison` exige que esse snapshot corresponda exatamente à regra habilitada e à
`legalVersion` local. Divergência retorna `sentence_rule_mismatch`. O fingerprint idempotente inclui
`chargeId`, e `mz_prison_sentences.charge_public_id` preserva o identificador canônico P6.

Isso mantém dois catálogos configuráveis por owners diferentes, mas elimina aceitação silenciosa de
divergência. Ambos devem ser preenchidos com os mesmos dados jurídicos aprovados; nenhum valor foi
preenchido nesta estabilização.

Commits isolados realizados:

- `mz_police` `a0ff0ce` — `fix(police): bind sentences to canonical charges`;
- `mz_prison` `7d1b3b8` — `fix(prison): persist canonical charge identity`;
- checkpoint anterior da continuidade em `mz_police`: `2ba865c`;
- auditoria obrigatória em `mz_docs`: `bb6d5f7`.

## Funcionalidades deliberadamente incompletas

| Item | Estado atual | Decisão desta estabilização |
|---|---|---|
| Associação de evidência a casos | `occurrenceAssociationEnabled = false`; os payloads P5 não aceitam `caseId`/`occurrenceId`. | Registrada como incompleta; nenhum vínculo P5→P6 foi inventado. |
| Coleta ambiental | Não há serviço/evento de coleta de vestígio no mundo. | Registrada como ausente; nenhum gameplay novo. |
| Trabalhos prisionais | `jobsEnabled = false` e `jobs = {}`; não há fluxo no service. | Mantido desativado e sem catálogo. |
| Destino automático de soltura | `releasePoints = {}` e o fluxo atual restaura propriedade/estado, sem teleporte. | Mantido sem destino inventado. |

## Consolidação Git

Todos os repositórios inspecionados permanecem em `main`. Os commits acima foram seguros porque
`mz_police` e `mz_prison` estavam limpos antes das correções.

Não foi criado commit em `mz_inventory`, `mz_org`, `mz_economy`, `mz_starter` ou `mz_testkit` porque as
mudanças policiais dependem de working trees maiores e preexistentes:

- `mz_inventory`: owner extraído ainda não commitado; integração policial/prisional usa
  `config.lua`, `shared/{constants,equipment,items}.lua`, `server/{core_bridge,schema,repository,
  service,police_access,exports,events,main}.lua` e seus harnesses. Separar apenas os adapters produziria
  um commit que não funciona sobre o HEAD `8627a2c`.
- `mz_org`: capabilities, seed e allowlist policiais dependem dos arquivos não rastreados
  `server/domain/*`; o HEAD é `e00eb75`.
- `mz_economy`: `server/accounts/police_charges.lua` depende da extração não commitada de accounts;
  o HEAD é `cf2c8b1`.
- `mz_starter`: boot, instaladores, preflights e documentação fazem parte de uma consolidação ampla já
  em andamento; o HEAD é `c356fcc`. As linhas policiais foram verificadas, mas não podem ser commitadas
  isoladamente sem carregar mudanças alheias do mesmo hunk/arquivo.
- `mz_testkit`: as oito fixtures corrigidas estão dentro da centralização ainda não commitada de testes;
  versioná-las sem registry/runner deixaria o checkpoint incompleto. O HEAD é `44f2c38`.

Essas alterações continuam necessárias no pacote efetivo de staging. Um deploy baseado apenas nos
commits HEAD dos cinco repositórios acima ficará incompleto.

## Manifesto de distribuição

### Recursos e revisões efetivas

| Resource | Versão | Commit base | Estado necessário no pacote |
|---|---:|---:|---|
| `mz_core` | 1.0.0 | `d595b39` | working tree atual; identidade/sessão/logs canônicos |
| `mz_economy` | 1.0.0 | `cf2c8b1` | working tree atual, incluindo `server/accounts/police_charges.lua` |
| `mz_menu` | 0.4.0-test | `b4cbfa9` | working tree atual; dependência de interação operacional |
| `mz_interact` | 0.2.0 | `bbd64d9` | working tree atual |
| `mz_vehicles` | 0.1.0 | `e44347f` | commit P4; somente `graphify-out` não rastreado |
| `mz_org` | 0.1.0 | `e00eb75` | working tree atual, incluindo `server/domain/*` e capabilities policiais |
| `mz_inventory` | 1.0.0 | `8627a2c` | working tree atual, incluindo owner extraído e adapters policial/prisional |
| `mz_fuel` | 1.0.0 | `942104b` | working tree atual; precede garagem |
| `mz_garagem` | 0.1.0 | `d5f5251` | working tree atual; readiness/garagem de serviço existente |
| `mz_police` | 0.7.0 | `a0ff0ce` | commit exato; diretório completo |
| `mz_prison` | 0.1.0 | `7d1b3b8` | commit exato; diretório completo |
| `mz_starter` | n/a | `c356fcc` | working tree atual; configuração e ferramentas de distribuição |

`mz_police` e `mz_prison` não possuem remote configurado. Para clean install, o pacote local deve
conter as pastas diretamente abaixo de `MZ_RESOURCE_SOURCE_ROOT`; o starter copia diretórios ausentes e
exclui `.git`, `graphify-out` e `node_modules`. Em instalação existente, o instalador preserva a pasta:
o operador deve publicar a revisão por seu processo normal de deploy.

Arquivos runtime indispensáveis:

- `mz_police/fxmanifest.lua`, todos os `client/*.lua` e `server/*.lua` declarados no manifest;
- `mz_prison/fxmanifest.lua`, `config.lua` e todos os `server/*.lua` declarados no manifest;
- os manifests e todos os arquivos por eles declarados dos owners listados acima;
- `mz_starter/cfg/resources.cfg`, `tools/install_deps.{bat,sh}` e
  `tools/check_structure.{bat,sh}` para instalação/preflight.

Testes, relatórios e `graphify-out` não são carregados pelo FXServer. Devem permanecer no pacote de
engenharia/evidência, mas podem ficar fora do diretório runtime da VPS.

### Dependências e ordem de boot

Ordem validada em `mz_starter/cfg/resources.cfg`:

`oxmysql/ox_lib → mz_core → mz_economy → mz_menu/mz_interact → mz_vehicles → mz_org → mz_inventory → mz_fuel → mz_garagem → mz_police → mz_prison`

Dependências diretas declaradas:

- `mz_police`: `oxmysql`, `mz_core`, `mz_org`, `mz_economy`, `mz_inventory`, `mz_vehicles`,
  `mz_interact`, `mz_menu`, `mz_garagem`;
- `mz_prison`: `oxmysql`, `mz_core`, `mz_inventory`.

`mz_prison` inicia depois de `mz_police`; a integração policial permanece desativada até a configuração
P7 ser aprovada. Os quatro exports prisionais aceitam somente o caller `mz_police`.

### Schemas necessários

Criados de forma idempotente pelos owners ao iniciar:

- P6: `mz_police_records`, `mz_police_cases`, `mz_police_case_officers`,
  `mz_police_case_people`, `mz_police_case_charges`, `mz_police_citations`,
  `mz_police_warrants`, `mz_police_history`;
- P7: `mz_prison_sentences`, `mz_prison_property`, `mz_prison_history`;
- owners: schemas atuais de `mz_core`, `mz_org`, `mz_inventory` e `mz_economy` presentes nos seus
  próprios preparers.

Para staging sem `mz_prison_sentences`, nenhum comando manual é necessário: o schema novo cria
`charge_public_id`. Para staging com tabela já existente:

```sql
SELECT COUNT(*) AS sentence_count FROM mz_prison_sentences;
SELECT COLUMN_NAME
FROM information_schema.COLUMNS
WHERE TABLE_SCHEMA = DATABASE()
  AND TABLE_NAME = 'mz_prison_sentences'
  AND COLUMN_NAME = 'charge_public_id';
```

Se a coluna não existir e `sentence_count = 0`, o operador pode, após backup e aprovação explícita,
executar a alteração aditiva:

```sql
ALTER TABLE mz_prison_sentences
  ADD COLUMN charge_public_id VARCHAR(160) NOT NULL AFTER case_public_id;
```

Se houver qualquer sentença existente, **parar**: não existe mapeamento seguro para inventar o
`charge_public_id`. Reconciliar os dados ou recriar a base descartável exige decisão humana. Nenhum SQL
foi executado nesta tarefa.

### Configurações obrigatórias e flags desativadas

Permanecem fail-closed por padrão:

- `MZPoliceConfig.Stations = {}` e catálogo do arsenal vazio;
- flags, motivos de impound e tipos de objetos viários vazios;
- `Evidence.occurrenceAssociationEnabled = false`;
- `Records.legalRulesEnabled = false`, `legalVersion = 'unapproved'`, jurisdições/catálogos vazios;
- `MZPoliceConfig.Prison.enabled = false`;
- `MZPrisonConfig.intakeEnabled = false`, `intakePoints = {}`;
- `sentenceRulesEnabled`, `reductionRulesEnabled`, `manualReleaseEnabled` e `jobsEnabled` em `false`;
- regras, motivos, jobs e `releasePoints` vazios;
- harness P1 removido do runtime na limpeza estrutural; QA preservado externamente.

O `dutyPoint` não pertence a `mz_police`. Um Staff autorizado deve usar o fluxo administrativo do
`mz_org`/`UpdateStaffOrganizationDutyPoint`, que persiste em `mz_orgs.config_json`:

```lua
dutyPoint = {
  enabled = true,
  coords = { x = <X_REAL>, y = <Y_REAL>, z = <Z_REAL> },
  radius = <1.0_A_5.0>,
  drawDistance = <5.0_A_50.0_E_NAO_MENOR_QUE_RADIUS>,
  label = '<LABEL_APROVADA>'
}
```

Coordenadas devem ser capturadas no servidor real. Sem esse dado,
`SetSelfOrgDutyAtPoint` retorna `duty_point_not_configured` e os testes operacionais de duty ficam
bloqueados. Nenhuma coordenada foi adicionada.

Antes de habilitar P7, os catálogos P6/P7 precisam usar a mesma jurisdição, `legalVersion`, código,
nome e duração aprovados, além de `intakePoints` reais. Só depois devem ser habilitados
`sentenceRulesEnabled` e `MZPoliceConfig.Prison.enabled`.

### Atualização e rollback

Atualização recomendada, sem deploy automático:

1. Abrir janela de manutenção sem jogadores, salvar backup do banco e dos diretórios atuais.
2. Montar uma release local única com os commits exatos e working trees requeridos acima; não usar
   somente `git clone` dos HEADs sujos.
3. Validar o pacote com os testes e preflights deste relatório.
4. Publicar para diretórios temporários na VPS e substituir os resources como uma unidade.
5. Fazer o preflight de schema; executar o `ALTER` apenas no caso vazio e aprovado descrito acima.
6. Iniciar na ordem canônica, confirmar readiness e manter todas as flags P6/P7 desativadas.
7. Executar G01–G10 e os cenários P2–P7 com sessões/personagens/MySQL reais; só então considerar
   habilitação gradual dos catálogos aprovados.

Rollback recomendado:

1. Parar `mz_prison` e `mz_police`, depois dependentes afetados, em ordem reversa.
2. Restaurar todos os diretórios do mesmo snapshot da release, não apenas um resource.
3. Reiniciar na ordem canônica e repetir readiness/smoke checks.
4. Não remover `charge_public_id`: versões anteriores ignoram a coluna aditiva. Se a release gravou
   dados e for necessário rollback integral, restaurar o backup do banco; não improvisar deletes.

## Evidências de validação

| Validação | Resultado |
|---|---|
| Runner global `mz_testkit -Suite covered` | **51/51 PASS** |
| `mz_police` P1–P7 | **13/13 arquivos PASS**; P7 prisão 12 casos, estático 9 |
| `mz_prison` P7 | **3/3 arquivos PASS**; service 14, prepare 2, estático 14 |
| Integrações selecionadas | inventory 4/4, org 1/1, economy 1/1, starter 9/9 PASS |
| Sintaxe Lua `mz_police` + `mz_prison` | PASS |
| `git diff --check` nos repositórios tocados | PASS; apenas avisos esperados LF/CRLF |
| Graphify | usado para descoberta; conclusões confirmadas nos fontes reais |
| FXServer + MySQL real | **NOT RUN / não aprovado** |

## Bloqueios e pendências de runtime

- As working trees essenciais de owners/starter/testkit ainda precisam ser consolidadas em uma release
  reproduzível sem absorver alterações preexistentes não relacionadas.
- `dutyPoint`, estações, intake e todos os catálogos operacionais/jurídicos dependem de valores reais e
  aprovação do operador.
- G01–G10, OneSync, sessões/personagens reais, retenção/restauração de inventário, transações MySQL,
  restart/reconnect e falhas de dependências só podem ser validados no FXServer.
- A presença de `schema_outdated_missing_charge_public_id` bloqueia `mz_prison` até a decisão de schema
  descrita acima.

Conclusão: toda estabilização estática tecnicamente segura foi concluída. **Não há PASS operacional** e
P8/P9 não foram iniciadas.
