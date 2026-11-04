---
title: MZ Framework — Visão arquitetural
status: CANONICAL
authority: LEVEL_2_ARCHITECTURE
owner: MZ Architecture
last_reviewed: 2026-10-07
applies_to: [all_resources, runtime_topology]
related_contracts: [MZ-RULES-v1, MZ-OWNERSHIP-v1, ADR-0002, ADR-0003, ADR-0016]
---

# Visão arquitetural

A MZ Framework é modular dentro de um único runtime FiveM. As fronteiras são de ownership, contrato e ciclo de vida; não representam microserviços.

## Camadas target

1. FiveM/OneSync e bibliotecas fundamentais.
2. `mz_core` como fundação mínima de account identity, character identity, session, lifecycle, autorização global e audit essencial.
3. Owners de domínio como `mz_org`, `mz_economy`, `mz_inventory`, `mz_vehicles`, `mz_houses` e `mz_population`.
4. Produtos/orquestradores como bank, garagem, fuel e admin.
5. Bridges externos opcionais.
6. Suporte não-runtime: `mz_docs`, futuro `mz_migrate` e futuro `mz_testkit`.

## CURRENT versus TARGET

Inventory foi extraído de `mz_core` para `mz_inventory` em `POST_INVENTORY_DOMAIN_EXTRACTION`. Organization foi extraído para `mz_org` em `POST_ORG_DOMAIN_EXTRACTION`. Vehicles foi extraído para `mz_vehicles` e Garage permanece em `mz_garagem` em `POST_VEHICLES_GARAGE_EXTRACTION`. Economy foi extraída para `mz_economy` em `POST_ECONOMY_SINGLE_WRITER_CUTOVER`. A validação runtime FiveM/MySQL pós-cutover continua pendente.

## Princípios

- um owner e um writer canônico por estado persistente;
- client payload nunca prova autoridade;
- products dependem dos owners que compõem;
- owner não depende de consumer;
- ausência opcional desabilita a capability relacionada de forma observável;
- state bags, caches e read models são projeções, não autoridade de mutação.

`mz_multichar` é um produto/orquestrador de seleção e criação. Ele possui somente a apresentação temporária da seleção: NUI, cena, câmera e preview peds locais. Preview peds não possuem session, Player State ou autoridade de personagem. O resource depende dos contratos de identity do `mz_core`; o core não depende de sua UI nem de seus internals.

`mz_population` possui somente o modelo virtual/estatístico de população. `mz_settings` mantém os controles físicos e ambientais GTA, incluindo densidade de peds/cenários, budgets e tráfego. A fronteira detalhada está no [contrato do domínio de população](POPULATION_DOMAIN.md).

A especificação detalhada permanece em `_architecture/mz_framework_v1/02_TARGET_ARCHITECTURE.md` e nos ADRs aceitos.
