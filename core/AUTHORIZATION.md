---
title: mz_core — Autorização global
status: CANONICAL
authority: LEVEL_2_ARCHITECTURE
owner: mz_core
last_reviewed: 2026-08-09
applies_to: [mz_core_staff, all_authorized_consumers]
related_contracts: [MZ-R024-MZ-R027, MZ-R050, MZ-R054-MZ-R057, ADR-0005, ADR-0015]
---

# Autorização global

## Modelo

O core responde à pergunta global: “o actor possui esta capability?”. O owner do domínio responde à pergunta contextual: “o actor pode agir sobre este agregado?”. Uma resposta global nunca ignora schema, lifecycle ou invariantes do owner.

Roles são conjuntos administráveis de capabilities. Consumers autorizam por capability explícita, não por nome ou nível de role. Membership em organização — inclusive uma organização legada chamada `staff` — não concede autorização global.

## ACE

ACE é trust input externo mapeado de forma explícita e namespaced. Resources pedem a decisão global ao core; não consultam ACE diretamente como substituto do contrato MZ.

## Cache e falha

Cache server-side de autorização é revisionado e invalidado em mudanças de role, grant, assignment ou session. Claim client-side é apenas projeção de UX. Capability desconhecida, cache stale, session inválida ou dependency failure resultam em deny.

## Auditoria

Mudanças de role/capability/assignment, overrides operacionais e decisões críticas negadas registram actor, capability, target, outcome e provenance sanitizada.

## Fontes

Detalhes da decisão: ADR-0005 e ADR-0015 em `_architecture/mz_framework_v1/adr/`. Contratos locais antigos de hierarquia/default roles foram classificados para reescrita e não são fontes normativas independentes.
