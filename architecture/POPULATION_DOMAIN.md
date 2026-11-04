---
title: MZ Framework — Population domain
status: CANONICAL
authority: LEVEL_2_ARCHITECTURE
owner: mz_population
last_reviewed: 2026-10-04
applies_to: [mz_population, mz_settings, population_consumers]
related_contracts: [MZ-OWNERSHIP-v1, MZ-R011-MZ-R023]
---

# Population domain

`mz_population` e o owner do modelo virtual e estatistico da populacao da cidade. O MVP 1 possui regions, zones, populacao agregada, workforce agregado, demanda normalizada e resolucao geografica por coordenadas.

## Boundary

| Responsabilidade | Owner |
|---|---|
| populacao virtual, regions/zones populacionais, workforce e demanda agregados | `mz_population` |
| densidade fisica de peds/cenarios, budgets, trafego, dispatch, wanted e controles ambientais GTA | `mz_settings` |
| identity, account, character, session e player lifecycle | `mz_core` |
| ledger e analytics financeiros passivos atuais | `mz_economy` |

`density` em uma zone de `mz_population` e um indicador estatistico entre `0.0` e `1.0`. Ele nao e nem altera `pedDensity`, `scenarioPedDensity`, vehicle density ou population budgets do GTA.

## Runtime and persistence

O estado canonico e server-side, construido de um bootstrap estatico validado e mantido somente em memoria. A API publica e read-only e retorna snapshots; nao existe referencia mutavel compartilhada com consumers.

O MVP nao cria schema ou repository porque nao possui mutacoes runtime que justifiquem persistencia. Tambem nao depende de jogador online, `mz_core`, `mz_settings` ou `mz_economy` para iniciar.

## Public consumption

Consumers futuros podem consultar regions, zones, population, workforce e demand pelos exports de `mz_population`. Business, jobs, economia ou sistemas ilegais continuam owners de suas proprias regras e nao devem ser incorporados ao dominio populacional.

## Deliberate exclusions

Nao existem NPCs fisicos, spawn/despawn, OneSync ownership, materializacao, IA, rotinas, pathfinding, consumo real, transacoes, precos, empresas ou empregos reais.

Uma eventual camada de materializacao entre dados virtuais e mundo GTA permanece decisao futura; o MVP nao atribui esse ownership a `mz_population` nem altera o boundary de `mz_settings`.
