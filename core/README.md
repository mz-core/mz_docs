---
title: mz_core — Resource documentation
status: CANONICAL
authority: LEVEL_2_ARCHITECTURE
owner: mz_core
last_reviewed: 2026-10-08
applies_to: [mz_core, core_consumers]
related_contracts: [MZ-CORE-BOUNDARY-v1, MZ-R001-MZ-R018, MZ-R024-MZ-R027, MZ-R048, MZ-R050, MZ-R054-MZ-R057]
---

# `mz_core`

## Purpose

Fundação mínima para bootstrap/readiness do core, identity, session, player lifecycle, estado fundamental necessário ao spawn, autorização global, contratos fundamentais e audit essencial.

## Owns

- account identity, character identity e identificadores fundamentais;
- session e vínculo explícito `source -> accountId -> activeCharacterId`;
- player lifecycle e estado fundamental necessário ao spawn seguro;
- roles/capabilities globais e assignments de staff;
- readiness do próprio core;
- audit writer essencial;
- baseline estrito das tabelas dessas capacidades.

## Does Not Own

Organization, economy, inventory, vehicles, garage, fuel, houses/property, phone product, HUD/presentation, security incidents/bans, admin UI, bridges externos, import/backfill, documentação oficial ou testes cross-resource. O core não cria, lê ou escreve saldos, itens, memberships, veículos, propriedades, conteúdo de telefone ou sanções.

## Dependencies

O manifest declara `oxmysql`, `ox_lib` e `spawnmanager`.

## Public API

Consulte o código atual e o catálogo histórico Wave 0 em `_architecture/mz_framework_v1/wave_0/catalog/apis.json`. Não há ainda catálogo vivo versionado em `mz_docs`; assinaturas não são duplicadas aqui.

Os contratos de identity/session atuais estão em [Public identity API](../api/IDENTITY_API.md) e o modelo em [Account, character and session](ACCOUNT_CHARACTER_SESSION.md).

## Events/Callbacks

Eventos de rede validam payload, session e actor server-side. Evento recebido não constitui autorização. Nomes e envelopes devem ser confirmados no código/catálogo antes de integração.

## Database

O core persiste apenas identity, session, autorização global, player lifecycle fundamental e audit essencial. `mz_accounts` representa Identity Account e não contém `wallet`, `bank` ou `dirty`; Financial Account pertence a `mz_economy`.

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
- `accountId` não é external identifier e `characterId` não é `citizenid`; `citizenid` é o identificador público canônico do Character;
- `playerLoaded` exige active character validado e vinculado à session;
- posição aceita para mutação crítica é plausível e server-observed;
- role code nunca é contrato de autorização de consumer;
- o core não adquire ownership por possuir código transitório de domínio.

## Architectural freeze

O boundary congelado, schema e procedimento de reset estao em [Core architectural freeze](CORE_ARCHITECTURAL_FREEZE.md).

## Integration

Veja o [guia de integração](../guides/CORE_INTEGRATION.md).

## Tests

Os harnesses do core ficam em `mz_testkit/tests/mz_core` e são executados pelo runner central com o working directory do resource. Harness offline não prova runtime FiveM/OneSync/MySQL.

## Detailed references

Contratos, operação, segurança, comandos e escopo detalhados estão no [índice de referências](reference/README.md).

## Related Rules

`MZ-R001`–`MZ-R018`, `MZ-R024`–`MZ-R027`, `MZ-R048`, `MZ-R050`, `MZ-R054`–`MZ-R057`.

## Related ADRs

ADR-0001, ADR-0002, ADR-0003, ADR-0005, ADR-0015 e ADR-0016.
