---
title: MZ Framework — Ownership e dependências
status: CANONICAL
authority: LEVEL_2_ARCHITECTURE
owner: MZ Architecture
last_reviewed: 2026-08-09
applies_to: [all_resources, database, public_contracts]
related_contracts: [MZ-R011-MZ-R023, MZ-R045-MZ-R048, MZ-R058-MZ-R070, ADR-0002, ADR-0016]
---

# Ownership e dependências

## Ownership target

| Domínio | Owner |
|---|---|
| identity, session, player lifecycle, global authorization, audit essencial | `mz_core` |
| organization, membership, grades, duty | `mz_org` |
| accounts, balances, ledger, transactions, payroll execution | `mz_economy` |
| items, containers, stashes, hotbar, drops | `mz_inventory` |
| vehicle identity, ownership, access, condition, persistent state, impound | `mz_vehicles` |
| garage policy/UI | `mz_garagem` |
| refuel gameplay/orchestration | `mz_fuel` |
| banking channel/UI | `mz_bank` |

## Dependency classes

- `REQUIRED`: ausência impede a capability principal.
- `OPTIONAL`: ausência desabilita apenas a integração relacionada.
- `FORBIDDEN`: direção viola boundary ou cria acoplamento indevido.
- `TEMPORARY_DURING_MIGRATION`: facade/shim com owner, prazo e remoção.

Nenhuma classificação permite SQL cross-owner como fallback. O grafo target deve permanecer acíclico; products/orchestrators ficam nas folhas e dependem dos owners.

## Estado atual

O snapshot Wave 0 é evidência histórica do grafo CURRENT. Para afirmações atuais, confira manifests e código. Para TARGET, use a arquitetura ratificada e os ADRs.
