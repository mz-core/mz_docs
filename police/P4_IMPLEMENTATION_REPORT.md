# MZ Police — relatório de implementação P4

Data: 2026-10-14  
Wave: P4 — fiscalização  
Estado: **gate estático aprovado; runtime FiveM pendente**

## Resultado

A P4 implementa uma camada de fiscalização server-authoritative sem transferir o ownership veicular para `mz_police`:

- ANPR manual sobre o veículo mais próximo;
- radar manual com velocidade derivada da entidade no servidor;
- flags persistidas em `metadata_json.policeFlags` pelo `mz_vehicles`;
- impound delegado ao fluxo canônico de `mz_vehicles`;
- objetos viários criados, rastreados, limitados e removidos pelo servidor;
- menu operacional via comando client `mz_police_tools` e `mz_menu`.

## Owner veicular

`mz_vehicles` ganhou três exports server-only para o chamador allowlisted `mz_police`:

- `PoliceScanNearestVehicle(actorSource, maximumDistance)`;
- `PoliceSetVehicleFlag(actorSource, scanId, flagCode, active, reference, operationId)`;
- `PoliceImpoundScannedVehicle(actorSource, scanId, reason, reference, operationId)`.

O owner deriva entidade, placa, distância, routing bucket e velocidade. O client não envia placa, net ID, velocidade ou estado. Scans expiram, ficam vinculados à sessão/personagem e são revalidados antes de mutações.

O fluxo policial autorizado reutiliza a mesma mutação, persistência, limpeza de world state e auditoria do impound existente. O export genérico anterior preserva suas regras de staff/ACE; não foi criado bypass administrativo.

## Configuração deliberadamente desativada

Permanecem vazios por padrão:

- `MZPoliceConfig.Enforcement.flags`;
- `MZPoliceConfig.Enforcement.impoundReasons`;
- `MZPoliceConfig.RoadObjects.types`.

Assim, nenhuma classificação policial, motivo legal, modelo de cone/barreira/spike strip ou decisão automática foi inventada. ANPR e radar também permanecem bloqueados até que as capabilities sejam atribuídas pelo owner `mz_org`.

As capabilities registradas, mas não adicionadas automaticamente ao seed policial, são:

- `police.impound`;
- `police.road.tools`;
- `police.anpr`;
- `police.radar`.

## Segurança

- policy de organização/capability/duty em toda operação;
- payloads fechados;
- rate limit por source;
- operation/request IDs idempotentes e com detecção de conflito;
- lock por placa no owner veicular;
- scan vinculado a source, session, citizenid, entidade e placa;
- distância e bucket reavaliados pelo `mz_vehicles`;
- objetos posicionados a partir de posição/direção server-side;
- limites por oficial e globais;
- cleanup no drop, perda de autoridade, troca de sessão e restart;
- logs correlacionáveis em `police` e `vehicles`;
- nenhuma punição automática ou confiança em state bag gravável pelo client.

## Arquivos da wave

### `mz_police`

- `config/server.lua`
- `server/bridge.lua`
- `server/audit.lua`
- `server/enforcement.lua`
- `server/events.lua`
- `server/custody.lua`
- `server/main.lua`
- `client/enforcement.lua`
- `fxmanifest.lua`
- `tests/p4_enforcement_harness.lua`
- `tests/p4_static_harness.lua`

### `mz_vehicles`

- `server/domain/service.lua`
- `server/domain/police.lua`
- `server/domain/exports.lua`
- `fxmanifest.lua`
- `tests/police_enforcement_contract_harness.lua`

### `mz_org`

- `shared/capabilities.lua`
- `server/domain/service.lua`
- `tests/police_capabilities_contract_harness.lua`

## Evidências estáticas

- regressão completa de `mz_vehicles`: 10 harnesses PASS;
- contrato policial de `mz_vehicles`: 7 casos PASS;
- `mz_police` P4: 8 casos PASS e contrato estático PASS;
- regressões P1–P3 de `mz_police`: PASS;
- allowlist/capabilities e regressões selecionadas de `mz_org`: PASS;
- sintaxe Lua e `git diff --check`: PASS no gate final da wave.

Os testes cobrem caller inválido, derivação de veículo/bucket, velocidade, flag, impound, replay, conflito, placa forjada, motivo não configurado, criação e remoção server-side de objeto. Mocks não equivalem ao runtime FiveM.

## Pendências de runtime

- atribuir capabilities aos cargos aprovados;
- cadastrar flags, motivos e objetos aprovados;
- validar OneSync natives de scan/criação/remoção de objeto no FXServer real;
- validar limpeza da entidade após impound e comportamento com streaming/routing buckets;
- validar UX do menu, ANPR e radar em veículos reais.

Nenhum runtime PASS é declarado.

## Próxima wave

P5 — evidências físicas e cadeia de custódia, preservando `mz_inventory` como owner dos itens.
