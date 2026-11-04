# MZ Police — relatório de implementação P3

Data: 2026-10-14  
Wave: P3 — delegacias e equipamentos  
Estado: **gate estático aprovado; runtime FiveM pendente**

## Resultado

A P3 adiciona pontos configuráveis de arsenal ao `mz_police`, preservando os owners já existentes:

- `mz_org`: membership, cargo, duty, capabilities e o ponto real de entrada/saída de serviço;
- `mz_inventory`: criação, UID, metadata, transação, equipamento e container restrito de devolução;
- `mz_garagem`: ponto, lista, spawn e recolhimento da frota de serviço;
- `mz_interact`: registro e lifecycle dos pontos configurados;
- `mz_menu`: menu de emissão e devolução;
- `mz_police`: policy, proximidade, catálogo autorizado e orquestração operacional.

Não existe chamada policial a `GiveWeaponToPed`, writer paralelo de item, ponto de duty duplicado ou spawn veicular próprio.

## Configuração deliberadamente vazia

`MZPoliceConfig.Stations` e `MZPoliceConfig.Armory.catalog` permanecem vazios. Assim, a instalação não cria ponto ou equipamento fictício.

Cada estação real deve ser adicionada em `mz_police/config/server.lua` com:

- chave da estação em minúsculas;
- `label`;
- `orgCode` presente em `AuthorizedOrganizations`;
- `armory.coords` com `x`, `y` e `z` medidos no servidor;
- opcionalmente `armory.drawDistance`, `armory.interactDistance`, `marker`, `blip` e `label`.

Cada item do catálogo deve conter:

- chave lógica em minúsculas;
- `item`, obrigatoriamente uma definição `unique = true` do `mz_inventory`;
- `label`;
- `minimumGrade`;
- opcionalmente `capability`; na ausência, usa `armory.basic`.

Nenhuma coordenada foi adicionada por esta wave.

### Duty point

O ponto de serviço não pertence à configuração do `mz_police`. Deve ser informado no fluxo administrativo de organizações do `mz_org`, que grava `config.dutyPoint` no owner com:

- `enabled = true`;
- `coords = { x, y, z }` reais;
- `radius` entre 1.0 e 5.0;
- `drawDistance` e `label` opcionais.

`SetSelfOrgDutyAtPoint` continua validando a posição server-side e falha com `duty_point_not_configured` enquanto esse dado não existir.

### Frota de serviço

`mz_garagem` já possui a garagem `police_service` e a frota `police_patrol`. A P3 não copiou esses dados nem criou uma API paralela. O acesso continua exigindo membership e duty no callback do próprio `mz_garagem`, e todo spawn passa por `mz_vehicles`.

As coordenadas atualmente presentes em `mz_garagem/shared/garages.lua` são configuração preexistente do owner e precisam ser confirmadas pelo operador no mapa; não foram criadas ou tratadas como evidência de runtime por esta wave.

## Contratos adicionados

Exports server-only de `mz_inventory`, allowlisted para `mz_police`:

- `ListPoliceIssuedEquipment(source, orgCode, stationId)`;
- `IssuePoliceEquipment(source, orgCode, stationId, catalogKey, itemName, operationId)`;
- `ReturnPoliceEquipment(source, orgCode, stationId, instanceUid, operationId)`.

Eventos de rede de `mz_police`:

- `mz_police:server:armory:open`;
- `mz_police:server:armory:issue`;
- `mz_police:server:armory:return`;
- resposta dirigida ao solicitante: `mz_police:client:armory:result`.

Os payloads são fechados. O cliente informa apenas estação, chave de catálogo, UID de uma devolução e request ID; nome real do item, organização, identidade, cargo, capability, duty e posição são resolvidos no servidor.

## Segurança e rastreabilidade

- autorização fail-closed por `armory.basic`, organização elegível e duty;
- `minimumGrade` e capability opcional por item;
- proximidade recalculada pelo servidor;
- rate limit por source;
- lock por source;
- replay idempotente e conflito de request/operation ID;
- somente itens `unique` podem ser emitidos;
- origem registrada em `metadata.policeIssue`;
- devolução exige o UID pertencente ao mesmo personagem, organização e estação;
- devolução move o item por transação para `police_armory`, preservando UID/metadata;
- equipamento/hotbar são reconciliados quando a devolução parte do loadout;
- logs policiais e do inventory usam operation ID correlacionável.

## Arquivos da wave

### `mz_police`

- `fxmanifest.lua`
- `config/server.lua`
- `server/bridge.lua`
- `server/audit.lua`
- `server/stations.lua`
- `server/events.lua`
- `server/main.lua`
- `server/custody.lua`
- `client/stations.lua`
- `tests/p1_harness.lua`
- `tests/p3_station_armory_harness.lua`
- `tests/p3_static_harness.lua`

### `mz_inventory`

- `config.lua`
- `shared/constants.lua`
- `server/service.lua`
- `server/police_access.lua`
- `server/exports.lua`
- `tests/police_armory_access_harness.lua`
- `tests/police_access_static_harness.lua`

### Documentação

- `mz_docs/police/P3_IMPLEMENTATION_REPORT.md`

## Evidências estáticas

- `mz_police` P1: 23 casos PASS;
- `mz_police` P2A: 16 casos PASS;
- `mz_police` P2B: 9 casos PASS;
- `mz_police` P2C: 6 casos PASS;
- `mz_police` P3 arsenal: 8 casos PASS;
- `mz_police` P3 contrato estático: PASS;
- `mz_inventory` grant policial: 11 casos PASS;
- `mz_inventory` arsenal policial: 8 casos PASS;
- contratos estáticos/owner/domain do inventory: PASS;
- contrato de garagem administrada e bootstrap de readiness: PASS.

Os harnesses cobrem autorização, cargo, distância, payload forjado, replay, conflito, allowlist de resource, organização inválida, emissão e devolução. Mocks não equivalem ao runtime FiveM.

## Pendências e limitações

- configurar coordenadas reais e catálogo aprovado;
- configurar e validar o `dutyPoint` no `mz_org`;
- validar pontos, marker, menu, inventário, equipamento e frota dentro do FiveM;
- validar restart/reconnect com MySQL real;
- confirmar as coordenadas preexistentes de `police_service` no mapa do servidor.

Nenhum runtime PASS é declarado.

## Próxima wave

P4 — fiscalização, mantendo `mz_vehicles` como autoridade de estado/impound e deixando valores operacionais não aprovados desabilitados.
