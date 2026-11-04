---
title: mz_core — Public identity read contracts
status: CANONICAL
authority: LEVEL_2_ARCHITECTURE
owner: mz_core
last_reviewed: 2026-10-08
applies_to: [server_consumers, identity, session]
related_contracts: [MZ-IDENTITY-v2, ADR-0018, MZ-R049, MZ-R050]
---

# Public identity read contracts

All contracts below are server exports of `mz_core`, return detached snapshots, and are read-only unless explicitly identified as orchestration-only.

| Export | Result | Failure/absence |
|---|---|---|
| `ResolvePlayerIdentity(source)` | `{ ok, source, accountId, characterId, citizenid, sessionId, active }` | typed `session_not_found`, `character_not_active`, or `player_not_loaded` |
| `GetAccountId(source)` | internal numeric account ID | `nil` without an active session |
| `GetActiveCharacterId(source)` | numeric `mz_players.id` | `nil` before activation |
| `GetActiveCharacter(source)` | character identity snapshot | `nil` before activation |
| `GetCharacterSnapshot(characterId)` | detached character identity snapshot | `nil` when absent |
| `GetCharactersForAccount(accountId)` | detached active-character roster | empty list when absent |
| `IsCharacterActive(source)` | boolean | `false` unless lifecycle is `CHARACTER_ACTIVE` |
| `GetPlayerSession(source)` | detached session snapshot | `nil` when absent |

`GetPlayerSnapshot` and `GetPlayerByCitizenIdSnapshot` include only Core-owned identity/lifecycle data: `accountId`, `characterId`, public `citizenid`, charinfo, position, fundamental state and session. They omit `license`, `job`, `gang`, money, inventory and other domain projections. No export returns the mutable player/session registry.

Mutation exports used by `mz_multichar` and `mz_creator` are caller-restricted and server-authoritative: `CreateCharacterDraft`, `ActivateCharacter`, `FinalizePendingCharacter`, and `CancelPendingCharacter`. They validate the implicit source session, ownership, slots, request ID, lifecycle and duplicate activation; client-supplied account identity is never accepted.

Consumers that keep runtime state by `source` must bind or revalidate `GetPlayerSession(source).id` and `activeCharacterId`, and clear state on disconnect, session replacement/source reuse and resource stop. A player snapshot alone is not runtime authority.
