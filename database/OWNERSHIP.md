---
title: MZ Framework — Ownership de banco de dados
status: CANONICAL
authority: LEVEL_1_NORMATIVE
owner: MZ Architecture
last_reviewed: 2026-08-09
applies_to: [all_persistent_resources, migrations]
related_contracts: [MZ-R011-MZ-R023, MZ-R052, MZ-R063-MZ-R070, ADR-0002]
---

# Ownership de banco de dados

Cada agregado persistente possui um owner e um writer canônico. O owner possui schema, migrations, invariantes e public contracts do domínio.

## Regras operacionais

- resource não escreve diretamente tabela de outro owner;
- leitura cross-domain usa API/read model; exceção SQL temporária é read-only, documentada e removível;
- cache, state bag ou shadow copy não adquire authority;
- operação cross-domain usa orchestration, idempotency e compensação explícita;
- migration cutover cerca, drena e desativa o writer antigo antes de ativar o novo;
- import/backfill executa em tooling explícito, nunca no startup normal.

## CURRENT versus TARGET

O `mz_core` ainda contém tabelas/repositories de org, economy, inventory e vehicles. Essa localização CURRENT não altera o ownership TARGET. Nenhuma tabela é movida na Wave 1A.

O catálogo Wave 0 em `_architecture/mz_framework_v1/wave_0/catalog/database.json` é um snapshot histórico. Catálogo vivo futuro deve ser extraído de migrations/repositories atuais, validado contra ownership e publicado de forma reproduzível.
