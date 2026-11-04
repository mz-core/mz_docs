---
title: mz_core - Architectural freeze
status: CANONICAL
authority: LEVEL_2_ARCHITECTURE
owner: mz_core
last_reviewed: 2026-10-08
applies_to: [mz_core, core_consumers]
related_contracts: [MZ-CORE-BOUNDARY-v1, MZ-IDENTITY-v2]
---

# Core architectural freeze

## Final ownership

`mz_core` owns:

- Identity Account and external Account identifiers;
- Character identity, Character lifecycle and public `citizenid`;
- ephemeral Session and the binding `source -> session -> account_id -> active_character_id`;
- player runtime lifecycle, fundamental position/spawn and Player State (`hunger`, `thirst`, `stress`, `health`, `armor`, death/respawn);
- global Staff roles, permissions and assignments;
- fundamental audit/logging and Core readiness/bootstrap.

`mz_core` does not own economy, inventory, organizations, vehicles, garage, houses/property, phone product, HUD presentation, appearance/clothing, security incidents or bans. Domain action names in the Player State gate and permission codes in the Staff catalog are authorization vocabulary, not domain persistence or writer authority.

## Final schema

The Core baseline creates exactly these tables:

- `mz_accounts`
- `mz_account_identifiers`
- `mz_players`
- `mz_player_sessions`
- `mz_character_requests`
- `mz_staff_roles`
- `mz_staff_role_permissions`
- `mz_staff_assignments`
- `mz_logs`

`mz_players.account_id` and `mz_player_sessions.account_id` are `NOT NULL`. Selection-time `character_id`, `active_character_id` and `citizenid` remain nullable. External identifiers exist only in `mz_account_identifiers`; `license` is not a Character or Session column. No speculative foreign keys were added because lifecycle code already controls creation/binding order and the existing deployment has no production data requiring an online migration.

The bootstrap performs `CREATE TABLE IF NOT EXISTS` followed by fail-fast checks for engine, charset, required columns/indexes, strict Account ownership, forbidden `license` columns and forbidden obsolete tables. It never performs `ALTER TABLE`, runtime backfill or schema conversion.

## RESET REQUIRED

An old local/staging database containing `mz_player_permissions`, `mz_schema_migrations`, Character/Session `license`, nullable Account ownership, or a different Core shape is intentionally rejected with `schema_reset_required`.

Stop the server, retain any diagnostic backup needed by the operator, recreate the disposable staging database/schema from the current resource baselines, and start from an empty MZ dataset. This repository does not ship or execute a compatibility migration, dual-read, dual-write or destructive automatic reset.

## Public identity and lifecycle

`source` and Session are ephemeral. `account_id` is the persistent Account identity; `character_id` is the internal persistent Character identity; `citizenid` is the canonical public Character identifier. Account resolution reads FiveM identifiers and persists them in `mz_account_identifiers`.

Character activation is explicit: Account Session -> selection/creation -> activation/finalization -> `loadCharacter` -> `CHARACTER_ACTIVE`. There is no `EnsurePlayerLoaded` or implicit legacy Character reconstruction. Session close has one lifecycle owner per path: drop/resource-stop closes through `MZIdentityService`; unloading Player runtime does not close Session.

## Presentation and integrations

Core owns spawn coordination, saved position and a minimal freemode bootstrap model. It calls only the native default-component reset required to make a newly created freemode ped usable. `mz_creator` owns persisted appearance and applies it after Core lifecycle events; Core does not persist or project `metadata.appearance`, choose clothing components or clear/apply props.

`mz_hud` consumes the canonical Player State event/snapshot. Core has no HUD service, HUD callback, `GetHUDState` export or `hudStateUpdated` adapter.

The optional direct `mz_banguard` spawn integration remains because model change and teleport leases must exist before the legitimate action to avoid a race. Core publishes the lifecycle-ready acknowledgment after spawn; BanGuard remains the sole owner of incidents, correlation and sanctions.

Medical-specific staging commands live in `mz_medical`, disabled by default and protected by its ACE/rate limit. Generic Player State staging remains in Core behind its convar and ACE.

## Mutation boundary

| Mutation group | Owner/caller | Required authority |
| --- | --- | --- |
| Character draft/activation | `mz_multichar` | caller allowlist, live Account Session, ownership, slots, lifecycle, request key |
| Character finalization/charinfo | `mz_creator` | exact caller, pending/active Character bound to source Session |
| Character phone field | `mz_phone` | exact caller and canonical `citizenid` |
| Player State/status/death/respawn | Core state service; configured domain writers | exact resource allowlist, loaded runtime identity, lifecycle/revision checks |
| Staff management | `mz_admin` | exact caller plus actor Account/Staff permission and hierarchy checks |

Read contracts remain unrestricted where they expose detached non-sensitive Core snapshots. Generic metadata mutation/read exports do not exist.

## Compatibility policy

The architectural freeze intentionally removes obsolete schemas, implicit load, identity migration gates, Character `license`, `job`/`gang` placeholders, generic metadata, HUD adapters and executable multichar backfill tooling. Historical checkpoint documents remain historical evidence only and do not describe the current runtime contract.
