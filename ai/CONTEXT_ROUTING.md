---
title: MZ Framework — AI context routing
status: REFERENCE
authority: LEVEL_4_REFERENCE
owner: MZ Documentation
last_reviewed: 2026-09-20
applies_to: [coding_agents, review_agents]
related_contracts: [MZ-DOCS-v1, MZ-R036, MZ-R039, MZ-R040]
---

# Context routing

| Sinal da pergunta | Carregar |
|---|---|
| player, account, character, multichar, identity, session, spawn, position | [core](../core/README.md), [account/character/session](../core/ACCOUNT_CHARACTER_SESSION.md), [player lifecycle](../core/PLAYER_LIFECYCLE.md), API/código atual |
| staff, ACE, capability, global role | [authorization](../core/AUTHORIZATION.md), [trust model](../security/TRUST_MODEL.md), ADR-0005/0015 |
| money, account, transfer, payroll | ownership/database, economy API/código; bank somente se channel/UI for afetado |
| org, grade, duty, membership | ownership, org API/DB; global authorization apenas se capability staff aparecer |
| item, stash, trunk, hotbar | inventory API/DB; owner contextual de house/vehicle quando aplicável |
| plate, impound, keys, condition | vehicles API/DB; garagem/fuel como callers/orchestrators |
| NetEvent, client payload, NUI | trust model + contrato do entrypoint + owner checks |
| schema, import, backfill | database ownership + owner migrations + migration ADR/tooling |
| dependency/start order | architecture overview + ownership/dependencies + manifests atuais |
| docs, source of truth, stale content | documentation standard + classification report da Wave 1A |

O router seleciona fontes; não decide arquitetura. Em conflito, compare authority/status/data, confira código CURRENT e registre a divergência.
