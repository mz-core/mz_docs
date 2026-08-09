---
title: MZ Framework — Visão arquitetural
status: CANONICAL
authority: LEVEL_2_ARCHITECTURE
owner: MZ Architecture
last_reviewed: 2026-08-09
applies_to: [all_resources, runtime_topology]
related_contracts: [MZ-RULES-v1, MZ-OWNERSHIP-v1, ADR-0002, ADR-0003, ADR-0016]
---

# Visão arquitetural

A MZ Framework é modular dentro de um único runtime FiveM. As fronteiras são de ownership, contrato e ciclo de vida; não representam microserviços.

## Camadas target

1. FiveM/OneSync e bibliotecas fundamentais.
2. `mz_core` como fundação mínima de identity, session, lifecycle, autorização global e audit essencial.
3. Owners de domínio como `mz_org`, `mz_economy`, `mz_inventory`, `mz_vehicles` e `mz_houses`.
4. Produtos/orquestradores como bank, garagem, fuel e admin.
5. Bridges externos opcionais.
6. Suporte não-runtime: `mz_docs`, futuro `mz_migrate` e futuro `mz_testkit`.

## CURRENT versus TARGET

O runtime atual ainda mantém responsabilidades de org, economy, inventory, vehicles, bridges e tooling dentro de `mz_core`. Isso é CURRENT e dívida de localização; não altera o owner TARGET. Extrações exigem waves próprias, single-writer cutover e testes.

## Princípios

- um owner e um writer canônico por estado persistente;
- client payload nunca prova autoridade;
- products dependem dos owners que compõem;
- owner não depende de consumer;
- ausência opcional desabilita a capability relacionada de forma observável;
- state bags, caches e read models são projeções, não autoridade de mutação.

A especificação detalhada permanece em `_architecture/mz_framework_v1/02_TARGET_ARCHITECTURE.md` e nos ADRs aceitos.
