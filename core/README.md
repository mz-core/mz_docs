---
title: mz_core — Resource documentation
status: CANONICAL
authority: LEVEL_2_ARCHITECTURE
owner: mz_core
last_reviewed: 2026-08-09
applies_to: [mz_core, core_consumers]
related_contracts: [MZ-CORE-BOUNDARY-v1, MZ-R001-MZ-R018, MZ-R024-MZ-R027, MZ-R048, MZ-R050, MZ-R054-MZ-R057]
---

# `mz_core`

## Purpose

Fundação mínima para bootstrap/readiness do core, identity, session, player lifecycle, estado fundamental necessário ao spawn, autorização global, contratos fundamentais e audit essencial.

## Owns

- identity e identificadores fundamentais do player;
- session e vínculo com a conexão;
- player lifecycle e estado fundamental necessário ao spawn seguro;
- roles/capabilities globais e assignments de staff;
- readiness do próprio core;
- audit writer essencial;
- tabelas e migrations dessas capacidades.

## Does Not Own

Organization, economy, inventory, vehicles, garage, fuel, banking UI, houses, admin UI, bridges externos, import/backfill, documentação oficial ou testes cross-resource. Parte desse código ainda está fisicamente no core no estado CURRENT e será extraída somente em waves autorizadas.

## Dependencies

O manifest declara `oxmysql` e `ox_lib`. O fluxo CURRENT de spawn também chama `spawnmanager`; essa dependência operacional deve permanecer explícita até futura correção de manifest autorizada.

## Public API

Consulte o código atual e o catálogo histórico Wave 0 em `_architecture/mz_framework_v1/wave_0/catalog/apis.json`. Não há ainda catálogo vivo versionado em `mz_docs`; assinaturas não são duplicadas aqui.

## Events/Callbacks

Eventos de rede validam payload, session e actor server-side. Evento recebido não constitui autorização. Nomes e envelopes devem ser confirmados no código/catálogo antes de integração.

## Database

O owner target do core persiste apenas identity, session, autorização global, player lifecycle fundamental e audit essencial. Tabelas de domínio ainda presentes no repository são dívida CURRENT, não ownership target.

## State

Cache, state bags e projeções client-side servem leitura/UX. Mutação crítica usa estado server-side observado ou persistido e vinculado à session ativa.

## Authorization

O core decide capability global. O owner do domínio decide autorização contextual e invariantes do agregado.

## Security

Client payload é não confiável; falhas de identity, session, authorization ou integrity são fail-closed. Veja o [trust model](../security/TRUST_MODEL.md).

## Failure Behavior

Readiness indisponível, session inválida, capability desconhecida ou dependência fundamental ausente devem produzir falha observável; operações críticas não fazem fallback permissivo.

## Invariants

- player/session são resolvidos server-side;
- `source` é locator de conexão, não identidade persistente;
- posição aceita para mutação crítica é plausível e server-observed;
- role code nunca é contrato de autorização de consumer;
- o core não adquire ownership por possuir código transitório de domínio.

## Integration

Veja o [guia de integração](../guides/CORE_INTEGRATION.md).

## Tests

Harnesses próximos permanecem em `mz_core/tests` durante a Wave 1A. Evidência cross-resource futura pertence a `mz_testkit`; harness offline não prova runtime FiveM/OneSync/MySQL.

## Related Rules

`MZ-R001`–`MZ-R018`, `MZ-R024`–`MZ-R027`, `MZ-R048`, `MZ-R050`, `MZ-R054`–`MZ-R057`.

## Related ADRs

ADR-0001, ADR-0002, ADR-0003, ADR-0005, ADR-0015 e ADR-0016.
