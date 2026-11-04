---
title: MZ Framework — Ownership e dependências
status: CANONICAL
authority: LEVEL_2_ARCHITECTURE
owner: MZ Architecture
last_reviewed: 2026-10-07
applies_to: [all_resources, database, public_contracts]
related_contracts: [MZ-R011-MZ-R023, MZ-R045-MZ-R048, MZ-R058-MZ-R070, ADR-0002, ADR-0016]
---

# Ownership e dependências

## Ownership target

| Domínio | Owner |
|---|---|
| account identity, character identity, session/active-character binding, player lifecycle, global authorization, audit essencial | `mz_core` |
| character roster/selection UX, temporary preview peds/scene/camera and creation orchestration | `mz_multichar` (product; no identity or spawn ownership) |
| organization, membership, grades, duty | `mz_org` |
| accounts, balances, ledger, transactions, payroll execution | `mz_economy` |
| items, containers, stashes, hotbar, drops | `mz_inventory` |
| clothing item custody/equipment slots | `mz_inventory`; `mz_clothing` owns garment catalog, shop purchase, outfits and visual projection |
| base face/body/hair appearance | `mz_creator` (does not own equipped clothing) |
| detecção, correlação, incidentes/evidência de segurança, bans e mitigação de hooks Cfx | `mz_banguard` |
| vehicle identity, ownership, access, condition, persistent state, impound | `mz_vehicles` |
| garage definitions, locations, policy, capacity and store/retrieve orchestration | `mz_garagem` |
| refuel gameplay/orchestration | `mz_fuel` |
| banking channel/UI | `mz_bank` |
| virtual regions/zones, aggregate population, workforce, demand and population geography resolution | `mz_population` — [contrato](POPULATION_DOMAIN.md) |
| GTA physical/ambient world controls, ped/scenario density, traffic and population budgets | `mz_settings` — [fronteira](POPULATION_DOMAIN.md) |
| custom animation assets, dictionaries, clip registry, retarget/export tooling | `mz_animations` — [contrato](CUSTOM_ANIMATIONS.md) |
| ledge detection, traversal rules, parkour state and world alignment | `mz_parkour`, consumidor de `mz_animations` |

## Dependency classes

- `REQUIRED`: ausência impede a capability principal.
- `OPTIONAL`: ausência desabilita apenas a integração relacionada.
- `FORBIDDEN`: direção viola boundary ou cria acoplamento indevido.
- `TEMPORARY_DURING_MIGRATION`: facade/shim com owner, prazo e remoção.

Nenhuma classificação permite SQL cross-owner como fallback. O grafo target deve permanecer acíclico; products/orchestrators ficam nas folhas e dependem dos owners.

## Estado atual

O snapshot Wave 0 é evidência histórica do grafo CURRENT. Para afirmações atuais, confira manifests e código. Para TARGET, use a arquitetura ratificada e os ADRs.

Vehicles e Garage atingiram o target em `POST_VEHICLES_GARAGE_EXTRACTION`; o contrato detalhado está em [Vehicles e Garage domain extraction](../vehicles/DOMAIN_EXTRACTION.md). Economy atingiu o single-writer target em `POST_ECONOMY_SINGLE_WRITER_CUTOVER`; o contrato está em [Economy direct extraction](../economy/DOMAIN_EXTRACTION.md).
