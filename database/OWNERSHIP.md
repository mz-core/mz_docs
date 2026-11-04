---
title: MZ Framework — Ownership de banco de dados
status: CANONICAL
authority: LEVEL_1_NORMATIVE
owner: MZ Architecture
last_reviewed: 2026-10-08
applies_to: [all_persistent_resources, migrations]
related_contracts: [MZ-R011-MZ-R023, MZ-R052, MZ-R063-MZ-R070, ADR-0002]
---

# Ownership de banco de dados

Cada agregado persistente possui um owner e um writer canônico. O owner possui schema, migrations, invariantes e public contracts do domínio.

`mz_core` é o owner de `mz_accounts`, `mz_account_identifiers`, da identidade de Character em `mz_players` e do binding em `mz_player_sessions`. External identifiers são credenciais vinculadas à Account; não são `accountId` e não são duplicados no Character. O baseline final é limpo e não possui backfill de players existentes.

## Regras operacionais

- resource não escreve diretamente tabela de outro owner;
- leitura cross-domain usa API/read model; exceção SQL temporária é read-only, documentada e removível;
- cache, state bag ou shadow copy não adquire authority;
- operação cross-domain usa orchestration, idempotency e compensação explícita;
- migration cutover cerca, drena e desativa o writer antigo antes de ativar o novo;
- import/backfill executa em tooling explícito, nunca no startup normal.

## CURRENT versus TARGET

O `mz_inventory` é o owner e writer canônico de `mz_inventory_items`, `mz_player_hotbar` e `mz_world_drops` desde `POST_INVENTORY_DOMAIN_EXTRACTION`. O `mz_org` é o owner e writer canônico das tabelas Organization listadas em `organizations/DOMAIN_EXTRACTION.md` desde `POST_ORG_DOMAIN_EXTRACTION`. O `mz_vehicles` é o owner e writer canônico de `mz_vehicles` e `mz_vehicle_runtime_state`; `mz_garagem` é o owner e writer de `mz_garage_definitions` desde `POST_VEHICLES_GARAGE_EXTRACTION`. `mz_economy` é o owner e writer canônico de `mz_player_accounts`, `mz_org_accounts`, `mz_account_idempotency`, `mz_org_account_operations`, `mz_economy_transactions` e stats derivadas desde `POST_ECONOMY_SINGLE_WRITER_CUTOVER`. O `mz_core` não cria, migra, lê nem escreve estado financeiro. A validação runtime FiveM/MySQL pós-cutover ainda está pendente.

O catálogo Wave 0 em `_architecture/mz_framework_v1/wave_0/catalog/database.json` é um snapshot histórico. Catálogo vivo futuro deve ser extraído de migrations/repositories atuais, validado contra ownership e publicado de forma reproduzível.
