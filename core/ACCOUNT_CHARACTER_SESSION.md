---
title: mz_core — Account, character and session foundation
status: CANONICAL
authority: LEVEL_2_ARCHITECTURE
owner: mz_core
last_reviewed: 2026-10-08
applies_to: [mz_core, mz_multichar, mz_creator, identity_consumers]
related_contracts: [MZ-IDENTITY-v2, ADR-0018, MZ-R001-MZ-R010, MZ-R066-MZ-R069]
---

# Account, character and session

The canonical runtime chain is:

```text
source -> session -> accountId -> activeCharacterId -> citizenid
```

- `source` is an ephemeral FiveM connection locator.
- `accountId` is the internal persistent account identity. A license, Discord ID or other external identifier is an account credential, never the account ID.
- `characterId` is the immutable numeric `mz_players.id` and the canonical character identity.
- `citizenid` is the stable canonical public identifier of a Character. It is distinct from the internal numeric `characterId`.
- `sessionId` identifies one connection session. A session has at most one active character.

## Persistence

`mz_core` owns `mz_accounts`, `mz_account_identifiers`, character identity in `mz_players`, and the account/character binding in `mz_player_sessions`. External identifiers exist only in `mz_account_identifiers`; `mz_players` and public Character DTOs do not duplicate `license`.

The unique active-character constraint is enforced server-side and by `mz_player_sessions.active_character_id`: active sessions may contain only one non-null value for a character. Closing or recovering a session clears that active marker.

## Lifecycle

```text
CONNECTED
  -> ACCOUNT_RESOLVED
  -> CHARACTER_SELECTION
  -> CHARACTER_CREATING (optional)
  -> CHARACTER_ACTIVATING
  -> CHARACTER_ACTIVE
  -> CHARACTER_UNLOADING
  -> DISCONNECTED
```

This identity lifecycle is separate from physical player state (`alive`, `downed`, `dead`, `respawning`). `mz_core:client:playerLoaded` is published only after the selected character belongs to the session account, the session binding is persisted, and the character runtime/state is initialized.

## Ownership

Account-scoped state includes external identifiers, account status and character-slot entitlement. Character-scoped Core state includes name, public `citizenid`, fundamental player state and position. Appearance belongs to `mz_creator`/clothing owners; org membership, money, inventory, vehicles, houses and phone content remain with their domain owners.

Global staff/admin authority is account-scoped. Persistent staff assignments use `account_id`; the active `character_id` may be recorded only as operational/audit context. Selecting another character on the same account must neither remove nor grant global staff authority.

## Public boundary

`GetPlayerSnapshot`, `GetPlayerByCitizenIdSnapshot` and `GetPlayerSession` return detached Core-owned data with `accountId`, `characterId` and `citizenid`. There is no implicit load API: activation follows Account Session -> Character selection -> activation/finalization -> `loadCharacter` -> `CHARACTER_ACTIVE`.

There is no runtime compatibility path, dual-read or identity backfill. New/staging databases must use the clean baseline.

## Switching

Hot character switching is `NOT YET SAFE`. The current runtime cannot prove complete cleanup of every character-scoped domain cache, state bag and projection. `ReturnToCharacterSelection` exists as a fail-closed contract and returns `hot_switch_not_safe`; reconnect-based selection is supported.

## Failure and recovery

- foreign-character activation is rejected without mutation;
- duplicate active-character sessions are rejected by the database-backed active marker;
- disconnect/resource-stop clears runtime caches and active markers;
- core restart marks stale sessions disconnected before accepting new sessions;
- creator cancellation archives the unfinalized draft and returns to selection;
- an activation failure after durable finalization restores the pending creator binding for retry/cancel;
- account-scoped mutation serialization prevents concurrent requests from consuming the same final slot;
- database and identity conflicts fail closed and are observable.
