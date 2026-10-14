# MZ Police — auditoria de continuidade P1–P7

Data: 2026-10-14  
Escopo: estado estático atual de `mz_police`, `mz_prison` e contratos owner consumidos pelas Waves P1–P7  
Resultado: **implementação estática das waves preservada, dois defeitos comprovados corrigidos, runtime FiveM/MySQL não aprovado**

## Base revisada

Foram lidos `P0_AUDIT_AND_PLAN.md`, os três documentos P1 (`P1_IMPLEMENTATION_REPORT.md`,
`P1_STAGING_VALIDATION_GATE.md`, `P1_FXSERVER_CHECKLIST.md`) e os relatórios de implementação P2–P7.
As afirmações foram confrontadas com manifests, fontes, contratos exportados, tabelas, histórico Git e testes atuais.
Graphify foi usado apenas para descoberta; as conclusões abaixo vêm dos arquivos reais e da execução dos testes.

## Resultado por wave

| Wave | Estado estático atual | Pendência operacional/produto |
|---|---|---|
| P1 | Policy, bridge, auditoria, readiness e harness console-only presentes. O harness permanece desativado por padrão e restrito a staging. | G01–G10 nunca foram executados com evidência no FXServer. `dutyPoint` real continua configuração externa do `mz_org`. |
| P2 | Custódia, colocação/retirada de veículo, revista por grant e apreensão delegada ao `mz_inventory` estão presentes e testadas. | Validar OneSync, sessões/personagens, queda/reconnect e inventário/MySQL reais. |
| P3 | Estações, arsenal e adapters de interação/menu estão presentes; garagem continua no owner existente. | `Stations = {}` e catálogo do arsenal vazio: o fluxo fica inerte até coordenadas e itens reais serem aprovados. |
| P4 | Scan, flags, impound e objetos viários estão presentes; mutação veicular permanece no `mz_vehicles`. | Flags, motivos de impound e tipos de objetos estão vazios; capabilities não são concedidas automaticamente. |
| P5 | Custódia e transferência de evidência no `mz_inventory` estão presentes. | Coleta ambiental não existe. `occurrenceAssociationEnabled` continua sem consumidor; a associação P5→P6 não foi concluída depois da P6. |
| P6 | Fichas, casos, pessoas, acusações, multas, mandados, histórico e liquidação econômica estão presentes. | Catálogo legal/jurisdições/mandados permanecem desativados e vazios; tabelas, ledger e reconciliação exigem MySQL/FiveM real. A NUI MDT não faz parte desta wave. |
| P7 | `mz_prison` independente, sentença, relógio online, redução, soltura e propriedade retida estão presentes. | Intake, regras, reduções, soltura manual e integração policial estão desativados. `releasePoints` e `jobs` são somente configuração reservada, sem fluxo implementado. |

Não foi encontrado código P8/P9, `mz_mdt`, `mz_dispatch` ou outro owner policial paralelo.

## Compatibilidade e duplicação

- Os exports consumidos por `mz_police` existem nos owners `mz_core`, `mz_org`, `mz_inventory`,
  `mz_vehicles`, `mz_economy` e `mz_prison`, com as assinaturas usadas pelos bridges.
- `mz_prison` depende diretamente apenas de `mz_core` e `mz_inventory`; seus quatro exports são
  server-side e aceitam exclusivamente o caller `mz_police`.
- Não há segundo writer Lua para as tabelas `mz_police_*` ou `mz_prison_*` fora dos respectivos owners.
- Todos os arquivos Lua de runtime de `mz_police` e `mz_prison` estão carregados por seus manifests;
  não foram encontrados módulos soltos nem registros públicos duplicados de eventos/exports.
- A duração da pena ainda possui dois catálogos operacionais (`Records.charges` e
  `MZPrisonConfig.sentenceRules`). A correção desta auditoria exige que exista uma acusação P6 com pena,
  mas não elimina divergência de duração se o operador configurar os dois catálogos de forma diferente.

## Defeitos comprovados e corrigidos

1. O adapter P7 aceitava qualquer `caseId` bem-formado e qualquer `ruleCode` configurado no prison.
   Agora a solicitação exige caso da organização autorizada e acusação com pena vinculada ao cidadão
   sob custódia e à mesma regra. O teste novo falhou antes da correção e passou depois.
2. P4 chamava exports de `mz_vehicles`, mas o manifest não declarava essa dependência direta.
   `mz_vehicles` foi adicionado às dependências e o contrato estático passou a impedir regressão.

Nenhuma funcionalidade foi reimplementada e nenhum schema/migração foi executado.

## Git e distribuição

- `mz_police`, `mz_prison`, `mz_inventory`, `mz_org`, `mz_vehicles`, `mz_economy`, `mz_garagem`,
  `mz_starter`, `mz_core`, `mz_docs` e `mz_testkit` estão em `main`.
- `mz_police` contém checkpoints P1/P2, P3, P4, P5, P6 e P7. `mz_prison` contém o checkpoint P7;
  `mz_vehicles` contém o checkpoint P4.
- As duas correções desta auditoria foram isoladas em `mz_police` no commit `2ba865c`.
- As extensões necessárias de P2/P3/P5/P7 em `mz_inventory`, as capabilities/configurações em
  `mz_org`, a liquidação P6 em `mz_economy` e as mudanças de distribuição em `mz_starter` continuam
  misturadas a working trees preexistentes e não estão integralmente commitadas.
- P0 e os documentos P1 continuam não rastreados em `mz_docs`; os relatórios P2–P7 estão commitados.
- `mz_police` e `mz_prison` não têm remote Git configurado. O instalador os trata como resources de
  distribuição local e exige `MZ_RESOURCE_SOURCE_ROOT`; sem empacotar/copiar esses diretórios, a VPS não
  receberá as waves.
- A ordem preparada no starter permanece
  `mz_core → mz_economy → mz_menu/mz_interact → mz_vehicles → mz_org → mz_inventory → mz_fuel → mz_garagem → mz_police → mz_prison`.

## Evidência de testes

- `mz_police`: **13/13 arquivos PASS** (P1–P7; P7 agora 11 casos).
- `mz_prison`: **2/2 arquivos PASS**.
- Contratos policiais/prisionais selecionados: `mz_inventory` 5/5, `mz_org` 4/4,
  `mz_vehicles`/`mz_garagem` 3/3 e `mz_economy`/`mz_starter` 2/2.
- Sintaxe Lua de `mz_police` + `mz_prison`: PASS.
- `git diff --check` nos arquivos corrigidos: PASS, com avisos de conversão LF/CRLF.
- Runner integrado `mz_testkit -Suite covered`: **43/51 PASS; 8 FAILED**. Portanto, o runner integrado
  não está aprovado. Quatro falhas usam fixture antiga sem `MZEquipment`; duas apontam para módulos já
  extraídos de Core (`server/vehicles/commands.lua` e `mz_economy/server/accounts/org_accounts.lua`);
  uma fixture de position security não carrega `MZIdentityService`; uma checagem externa acusa ausência
  de guard na clothing. Essas falhas não atravessam os fluxos P1–P7 testados e não foram alteradas nesta auditoria.

## Veredito

O código P1–P7 está coerente com os relatórios no que eles declaram como implementação estática, com as
exceções/pendências explicitadas acima. O gate estático específico está aprovado após as duas correções.
O conjunto ainda **não está operacionalmente aprovado**: configurações reais, distribuição dos working
trees, banco, OneSync, sessões reais, restart/reconnect e o gate G01–G10 dependem exclusivamente de execução
documentada dentro do FiveM/FXServer.
