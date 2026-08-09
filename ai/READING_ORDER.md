---
title: MZ Framework — AI reading order
status: REFERENCE
authority: LEVEL_4_REFERENCE
owner: MZ Documentation
last_reviewed: 2026-08-09
applies_to: [coding_agents, review_agents]
related_contracts: [MZ-DOCS-v1, MZ-R036, MZ-R039, MZ-R040, MZ-R053]
---

# Reading order

## Mudança em um resource

`AI_CONTEXT` → rules relevantes → `RESOURCE_INDEX` → doc do owner → API/DB/security específicos → ADRs → código e testes próximos → consumers conhecidos.

## Mudança cross-resource

Acrescente ownership/dependencies, contratos de comunicação e todos os consumers conhecidos. Verifique ciclos e single-writer.

## Incidente

Acrescente failure behavior, operations, observability, deployment profile e evidência recente. Relatório histórico é hipótese, não diagnóstico atual.

## Migração ou compatibilidade

Acrescente owner target, consumer inventory, ADR, plano de dados, fence/drain/cutover, reconciliação e rollback. Não trate shim como writer alternativo.

## Documentação

Acrescente o [standard documental](../standards/DOCUMENTATION_STANDARD.md), confirme source of truth e valide links/metadata. Não carregue catálogos completos quando um índice resolver a pergunta.
