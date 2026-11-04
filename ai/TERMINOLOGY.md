---
title: MZ Framework — Terminologia canônica
status: CANONICAL
authority: LEVEL_2_ARCHITECTURE
owner: MZ Architecture
last_reviewed: 2026-09-20
applies_to: [all_resources, documentation, coding_agents]
related_contracts: [MZ-RULES-v1, ADR-0001, ADR-0002, ADR-0015, ADR-0017]
---

# Terminologia

- **resource**: unidade FiveM com manifest e lifecycle próprios.
- **owner**: responsável único por invariants, writer, schema, migrations e public contract de um domínio.
- **canonical writer**: único caminho autorizado a persistir determinado estado.
- **caller**: resource/código que invoca um contrato.
- **actor**: identidade em cujo nome a ação ocorre, resolvida server-side.
- **authority**: fonte capaz de confirmar identity, state ou permission.
- **capability**: permissão global explícita avaliada pelo core; não é role code.
- **contextual authorization**: decisão do owner sobre actor e agregado/contexto específicos.
- **accountId**: identidade persistente interna da conta; não equivale a license/Discord ID.
- **characterId**: identidade persistente canônica de personagem, atualmente `mz_players.id`.
- **session**: vínculo temporal server-side `source -> accountId -> activeCharacterId`.
- **citizenId**: identificador legado estável de personagem durante a migração; não equivale a `source` nem `accountId`.
- **projection**: cópia para leitura/UX sem autoridade de mutação.
- **CURRENT**: comportamento/localização observado no runtime e código atuais.
- **TARGET**: boundary/ownership ratificado a alcançar por migrations autorizadas.
- **bridge**: adaptador externo que traduz contratos de outra framework para APIs MZ.
- **migration**: processo explícito de dados/contratos com cutover, reconciliação e rollback.
- **legacy**: código ou contrato anterior identificado; não significa seguro, suportado ou canônico.
- **runtime evidence**: execução no ambiente declarado; diferente de static scan ou harness.

Sinônimos legados devem apontar ao termo canônico e declarar escopo temporal.
