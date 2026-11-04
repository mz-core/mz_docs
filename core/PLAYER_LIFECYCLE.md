---
title: mz_core — Player lifecycle
status: REFERENCE
authority: LEVEL_4_REFERENCE
owner: mz_core
last_reviewed: 2026-09-20
applies_to: [mz_core_player, player_consumers]
related_contracts: [MZ-CORE-BOUNDARY-v1, MZ-R001-MZ-R010, ADR-0001]
---

# Player lifecycle

## Escopo

O core mantém identity, session, registry online, transições de lifecycle e estado fundamental necessário ao spawn. Estado específico de feature — duty, saldo, items, vehicle condition ou condição médica — pertence ao respectivo owner.

## Autoridade

- `source` identifica a conexão ativa e deve ser resolvido para a session/player server-side;
- `accountId` é a identidade persistente interna da conta;
- `characterId` (`mz_players.id`) é a identidade canônica do personagem;
- `citizenId` é o identificador legado estável do personagem durante a migração;
- state bags e payloads client-side são projeções/inputs não confiáveis;
- posição usada por mutação crítica deve ser server-observed, plausível e vinculada à session atual.

## Integração

Antes de consumir estado do player, confirme readiness e session. Use contratos públicos observados no código; não leia tabelas do core ou caches internos. Consumers devem tolerar unload, reconnect, source reuse e falha intermediária.

O lifecycle de identidade passa por `CHARACTER_SELECTION`, `CHARACTER_ACTIVATING` e `CHARACTER_ACTIVE`. O evento `mz_core:client:playerLoaded` somente ocorre no último estado. Veja [Account, character and session](ACCOUNT_CHARACTER_SESSION.md).

## Evidência

Os documentos locais `PLAYER_STATE_SERVER_CONTRACT.md`, `PLAYER_STATE_CLIENT_SYNC.md`, `PLAYER_STATE_INTEGRATION_GUIDE.md` e `PLAYER_STATE_SECURITY.md` foram fontes da reescrita, confrontadas com o código CURRENT e a arquitetura ratificada. Seus worklogs/resultados datados não foram promovidos como contrato oficial.
