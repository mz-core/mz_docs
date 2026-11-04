---
title: Inventory Phase 1 Backend Foundation
status: SUPERSEDED_BY_DIRECT_DOMAIN_EXTRACTION
authority: LEVEL_4_REFERENCE
owner: mz_inventory
last_reviewed: 2026-10-02
applies_to: [mz_inventory, inventory, typed_owners]
related_contracts: [ADR-0007, ADR-0019]
---

# Inventory Phase 1 backend foundation

> Historical baseline only. The shadow foundation was promoted and replaced by the direct single-writer extraction documented in [DOMAIN_EXTRACTION.md](./DOMAIN_EXTRACTION.md). Its read-only exports, bridge plan and compatibility normalizer are no longer active code.

`mz_inventory` now contains a server-side foundation, but `mz_core` remains the sole canonical Inventory writer.

## Active boundary

```text
source
  -> mz_core ResolvePlayerIdentity
  -> mz_core GetPlayerSession
  -> require CHARACTER_ACTIVE and matching source/session/account/row id
  -> canonical owner { ownerType='character', ownerId=citizenid }
  -> optional read-only mapping to CURRENT storage
```

The server never accepts a client-supplied owner as authority. An expected owner id is only a candidate and must equal the server-resolved `citizenid`. `source` is runtime context and is never a persistent owner.

## Typed owners

| Owner type | Owner id |
|---|---|
| `character` | canonical `citizenid` |
| `vehicle` | stable vehicle-domain identity |
| `property` | `property_id`/stable property identity |
| `organization` | `org_id`/stable organization identity |
| `world_drop` | server-generated drop id |
| `temporary` | runtime-only id; persistence is rejected |

Compatibility normalization recognizes CURRENT `player`, `org`, `house`, `world` and `stash + personal:<citizenid>` only when legacy mode is explicitly requested. Canonical callers cannot silently use those aliases.

## Phase 1 APIs

The following server exports are read-only/foundation contracts:

- `GetInventoryBackendFoundationStatus()`;
- `NormalizeInventoryOwnerDescriptor(descriptor, allowLegacy)`;
- `ResolveInventoryCharacterOwner(source, expectedOwnerId)`;
- `ReadInventoryShadow(source, expectedOwnerId, containerType)`;
- `GetInventorySchemaOwnershipPlan()`;
- `InspectInventorySchemaReadOnly()`;
- `GetInventoryLegacyBridgePlan()`.

No network event or NUI callback was added. No mutation export exists in `mz_inventory` yet.

## Single-writer guarantee

Phase 1 reports `mode=shadow_read_only`, `canonicalWriter=mz_core`, `writerEnabled=false` and `schemaApplyEnabled=false`. Schema apply returns `schema_ownership_not_promoted`; bridge write forwarding returns `single_writer_flip_not_promoted`; service write attempts return `inventory_writer_not_promoted`.

The live client bridge continues calling `mz_core:server:inventory:*`. The tables `mz_inventory_items`, `mz_player_hotbar` and `mz_world_drops` are still created/repaired by `mz_core/server/prepare.lua` and mutated only by the core Inventory backend.

## What remains

Before a later single-writer flip: complete mutation parity, validate schema/clean install, add read comparisons in staging, preserve locks/reservations/hotbar/weapons/drops/transfers, prepare owner-type preview, migrate consumer batches, and prove A/B plus A/B/A in real FiveM/OneSync/MySQL runtime.

Static harnesses are not runtime proof. The preserved pre-cutover A/B evidence remains the current runtime baseline.

## Reference-pattern backlog boundary

The local OX Inventory reference was reviewed for useful lifecycle concepts. The architecture backlog now places server-side vehicle proof, inventory view leases, shared-view cleanup, lock-release regressions, filtered hooks and delta synchronization in their correct later phases.

None belongs to the current Phase 1 VPS gate. The current gate remains limited to proving the read-only foundation and confirming that `mz_core` is still the only production writer. The complete routing and the patterns that must not be ported are documented in `_architecture/mz_framework_v1/identity_consumers/INVENTORY_REFERENCE_PATTERN_AUDIT.md`.
