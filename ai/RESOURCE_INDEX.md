---
title: MZ Framework — AI resource index
status: REFERENCE
authority: LEVEL_4_REFERENCE
owner: MZ Documentation
last_reviewed: 2026-08-09
applies_to: [resource_routing, coding_agents]
related_contracts: [MZ-OWNERSHIP-v1, MZ-R011-MZ-R023, MZ-R045, MZ-R058]
---

# Resource index

Este índice roteia contexto; não substitui documentação do resource nem o código. A classificação usa arquitetura ratificada e o snapshot Wave 0. `UNKNOWN` significa “não verificado”, não ausência de responsabilidade.

| Resource | Owner/responsabilidade target | Documentação | API/catalog reference |
|---|---|---|---|
| `mz_core` | identity, session, player lifecycle, global authorization, audit essencial | [core](../core/README.md) | `wave_0/catalog/apis.json` + código atual |
| `mz_org` | organizations, memberships, grades, duty | [ownership](../architecture/OWNERSHIP_AND_DEPENDENCIES.md) | Wave 0 + código atual |
| `mz_economy` | accounts, balances, ledger, transactions, payroll execution | [database ownership](../database/OWNERSHIP.md) | Wave 0 + código atual |
| `mz_inventory` | items, containers, stashes, hotbar, drops | [ownership](../architecture/OWNERSHIP_AND_DEPENDENCIES.md) | Wave 0 + código atual |
| `mz_vehicles` | vehicle identity, ownership, access, condition, persistent state, impound | [ownership](../architecture/OWNERSHIP_AND_DEPENDENCIES.md) | Wave 0 + código atual |
| `mz_houses` | property/access domain | [ownership](../architecture/OWNERSHIP_AND_DEPENDENCIES.md) | Wave 0 + código atual |
| `mz_bank` | banking channel/UI; não possui ledger | [ownership](../architecture/OWNERSHIP_AND_DEPENDENCIES.md) | Wave 0 + código atual |
| `mz_garagem` | garage policy/UI; não possui persistent vehicle state | [ownership](../architecture/OWNERSHIP_AND_DEPENDENCIES.md) | Wave 0 + código atual |
| `mz_fuel` | refuel gameplay/orchestration; não possui persistent vehicle condition | [ownership](../architecture/OWNERSHIP_AND_DEPENDENCIES.md) | Wave 0 + código atual |
| `mz_admin` | administrative workflows/UI; usa global capabilities e owner checks | [authorization](../core/AUTHORIZATION.md) | Wave 0 + código atual |

## Resources observados sem owner target verificado

`mz_animations`, `mz_assets`, `mz_banguard`, `mz_chat`, `mz_clothing`, `mz_creator`, `mz_doors`, `mz_feedback`, `mz_furniture`, `mz_hud`, `mz_interact`, `mz_interiors`, `mz_lib`, `mz_loading`, `mz_medical`, `mz_menu`, `mz_notify`, `mz_org_activities`, `mz_parkour`, `mz_phone`, `mz_phone_server`, `mz_progress`, `mz_props`, `mz_radio`, `mz_realestate`, `mz_settings`, `mz_starter`, `mz_status`, `mz_sync`, `mz_target`, `mz_tatto`, `mz_warns`, `mz_whitelist` e `mz_world_editor`.

Para esses resources, consulte manifest/código atuais e `_architecture/mz_framework_v1/wave_0/catalog/resources.json`; não inferir owner a partir do nome.
