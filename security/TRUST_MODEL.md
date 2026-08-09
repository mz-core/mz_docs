---
title: MZ Framework — Trust model
status: CANONICAL
authority: LEVEL_1_NORMATIVE
owner: MZ Security
last_reviewed: 2026-08-09
applies_to: [all_network_entrypoints, public_mutations]
related_contracts: [MZ-R001-MZ-R010, MZ-R024-MZ-R027, MZ-R033, MZ-R050-MZ-R057, ADR-0001, ADR-0015]
---

# Trust model

## Fronteiras

- Client, NUI, state bag e payload de NetEvent são não confiáveis.
- O `source` implícito do runtime inicia a resolução; não prova identidade persistente.
- Identity, session, authorization e estado crítico são resolvidos/revalidados server-side.
- Eventos representam transporte ou fatos; sua existência não concede autorização.

## Mutação pública

Toda mutação declara caller, actor, schema, limits, owner, autorização global quando aplicável, checks contextuais do domínio, failure behavior, idempotência e audit. O owner rejeita campos/valores inválidos e falha fechado.

## Projeções

State bags, caches e read models podem acelerar leitura e UX. Eles possuem produtor e política de consistência, mas nunca substituem o writer canônico.

## Segredos e observabilidade

Logs/audits não registram secrets, credentials ou authorization headers. Evidência deve diferenciar análise estática, harness e runtime real.
