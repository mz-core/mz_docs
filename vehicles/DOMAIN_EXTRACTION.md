---
title: MZ Framework — Vehicles e Garage domain extraction
status: CANONICAL
authority: LEVEL_1_NORMATIVE
owner: MZ Architecture
last_reviewed: 2026-10-07
applies_to: [mz_core, mz_vehicles, mz_garagem, mz_org, mz_fuel, database]
related_contracts: [MZ-OWNERSHIP-v1, POST_VEHICLES_GARAGE_EXTRACTION]
---

# Vehicles e Garage domain extraction

## Boundary final

`mz_vehicles` é o owner canônico do registro de veículo, `vehicle_id`, placa, modelo, ownership, propriedades, condição, estado persistente (`stored`, `out`, `impounded`), associação lógica à garagem, dados de impound e associação entre registro persistente e entidade GTA.

`mz_garagem` é o owner das definições de garagem, localização, filtros, permissões locais, capacidade e orquestração de store/retrieve/release. Ele valida a garagem e chama `mz_vehicles`; nunca escreve tabelas de Vehicle.

`mz_core` mantém somente identidade de Account/Character, session/lifecycle, autorização global e audit. Não carrega código, schema, repository, events ou exports de Vehicle/Garage.

## Identidade e ownership

- `mz_vehicles.id` é o `vehicle_id` persistente e a identidade canônica.
- `plate` é um atributo único e um lookup operacional; não é a PK.
- `owner_type = 'player'` usa `owner_id = citizenid`; troca de `source` ou session não altera ownership.
- `owner_type = 'org'` usa o ID persistente da Organization. `mz_org` valida membership e concede ao domínio Vehicle um grant runtime vinculado ao Character ativo.
- `net_id` e `entity_handle` são associações runtime, nunca ownership ou identidade persistente.

## Modelo persistente

| Tabela | PK | FKs | Índices importantes | Writer |
|---|---|---|---|---|
| `mz_vehicles` | `id` | nenhuma | `UNIQUE plate`, `(owner_type, owner_id)`, `(garage, state)` | `mz_vehicles` |
| `mz_vehicle_runtime_state` | `vehicle_id` | `vehicle_id -> mz_vehicles.id ON DELETE CASCADE` | `UNIQUE plate`, `state`, `last_seen_at` | `mz_vehicles` |
| `mz_garage_definitions` | `id` | nenhuma | `UNIQUE garage_code`, `(kind, active)`, `(org_code, active)` | `mz_garagem` |
| `mz_garage_service_state` | `(citizenid, garage_id, entry_id)` | nenhuma | `(garage_id, fleet_id, entry_id)` | `mz_garagem` |

`garage` e `state` permanecem no agregado Vehicle porque respondem onde o veículo persistente está e se ele pode existir no mundo. `mz_garagem` possui as regras que autorizam a transição, mas a mutação é executada somente por `mz_vehicles`.

`mz_garage_service_state` guarda somente a condição de unidades temporárias da frota de serviço por personagem, garagem e entrada. Ele não cria um veículo persistente nem substitui `mz_vehicles`; o runtime de serviço continua efêmero e a entidade GTA continua sob `mz_vehicles`.

`impound_data` acompanha o lifecycle do veículo e também é escrito por `mz_vehicles`. `mz_garagem` decide se a garagem atual permite release; permissões staff continuam no `mz_core` como autorização global.

## Runtime e spawn

O caller (`mz_garagem`) valida interação, localização e política. `mz_vehicles` valida registro, ownership/acesso e lifecycle, autoriza a retirada, registra o snapshot e mantém o mapeamento runtime. O client de `mz_vehicles` cria/restaura a entidade GTA. Reinício de resource ou troca de session limpa caches por `source`; o registro continua identificado por `vehicle_id`.

## Contratos públicos

### `mz_vehicles`

Reads: `GetVehicleById`, `GetVehicleByPlate`, `GetPlayerVehicles`, `GetPlayerVehiclesByCitizenId`, `GetVehiclesByOwner`, `GetAccessibleVehicles`, `PlayerOwnsVehicle`, `OwnerOwnsVehicle`, `CanAccessVehicle`, `GetOutVehiclesForRespawn`, `GetOutVehiclesNearCoords`, `IsPlateSpawned`, `IsVehicleDomainReady`.

Mutations: `RegisterPlayerVehicle`, `RegisterVehicleForOwner`, `SetVehicleStored`, `SetVehicleGarage`, `SetVehicleState`, `SetVehicleMetadata`, `SetVehicleProps`, `SetVehicleCondition`, `TakeOutVehicle`, `StoreVehicle`, `ImpoundVehicle`, `ReleaseImpoundVehicle`, `RegisterOutVehicleEntity`, `UpdateOutVehicleSnapshot`, `MarkOutVehicleDestroyed`.

Access/runtime: `GrantVehicleOwnerAccess` (somente `mz_org`), `EnsureVehicleAccessForPlayer`, `GrantVehicleAccess`, `RevokeVehicleAccess`, `HasVehicleAccess`, `RevokeAllVehicleAccess`, `RestoreWorldVehiclesForPlayer`, `EnsureWorldVehiclesNearPlayer`.

Catálogo: `GetCatalogEntry`, `ResolveCatalogEntry`, `ResolveImageKey`, `ResolveVehicleStorage`, `GetBlockedRule`, `IsModelBlocked`.

### `mz_garagem`

`OpenHouseGarage`, `IsManagedGarageReady`, `CreateManagedGarage`, `ListManagedGaragesForAdmin`, `UpdateManagedGarage`, `SetManagedGarageActive`, `ListManagedGarages`.

Store/retrieve são callbacks server-authoritative de `mz_garagem`; a escrita final passa pelos exports de `mz_vehicles`.

## Dependências

```text
mz_core
   ↑
mz_vehicles ← mz_org
   ↑           ↑
mz_fuel     mz_garagem
               ↑
            mz_admin
```

`mz_vehicles` depende de `mz_core` para Character/session, autorização staff e audit. `mz_org`, `mz_fuel` e `mz_garagem` dependem de `mz_vehicles`. Não existe dependência `mz_vehicles -> mz_org` ou `mz_vehicles -> mz_garagem`; portanto não há ciclo.

## Cutover de pré-produção

Nenhuma migration de compatibilidade foi criada porque a MZ Framework ainda está em pré-produção. O baseline novo cria `mz_vehicles` e `mz_vehicle_runtime_state`; `mz_player_vehicles` e `mz_vehicle_world_state` não possuem consumers.

Para limpar um banco local antigo, pare o servidor, descarte explicitamente as tabelas antigas e reinicie os resources para recriar o baseline. Essa operação é destrutiva e não é executada automaticamente pelo resource nem por tooling desta wave.
