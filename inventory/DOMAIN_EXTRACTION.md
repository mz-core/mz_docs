---
title: Inventory Domain Extraction
status: IMPLEMENTED_STATIC_RUNTIME_PENDING
authority: LEVEL_4_REFERENCE
owner: mz_inventory
last_reviewed: 2026-10-02
applies_to: [mz_inventory, mz_core, inventory]
related_contracts: [ADR-0019]
---

# Inventory domain extraction

`mz_inventory` is the sole Inventory schema, repository, service and writer owner. `mz_core` supplies only active identity/session/player-state and domain authorization contracts.

## Identity and owners

```text
Character    character / citizenid
Vehicle      vehicle / vehicle_id
Property     property / property_id
Organization organization / org_id
World drop   world_drop / drop_uid
Temporary    runtime only, never persisted
```

`citizenid` remains the canonical persistent Character identifier. Client payloads never establish persistent ownership.

## Startup

`mz_core` must start before `mz_inventory`; all Inventory consumers start afterward. The canonical starter order already enforces this.

At startup, `mz_inventory` creates and owns `mz_inventory_items`, `mz_player_hotbar` and `mz_world_drops`. It fails closed when legacy owner rows are present.

## Staging reset

No reset is automatic. With the server stopped and the database confirmed as the disposable `mz_starter` staging database:

```sql
USE `mz_starter`;
DROP TABLE IF EXISTS `mz_player_hotbar`;
DROP TABLE IF EXISTS `mz_inventory_items`;
DROP TABLE IF EXISTS `mz_world_drops`;
```

Restart from `oxmysql`/core/inventory and verify the log:

```text
[mz_inventory][schema] ready writer_owner=mz_inventory single_writer=true
[mz_inventory] ready mode=canonical_writer owner=mz_inventory
```

## Integration

Consumers call exports and callbacks on `mz_inventory`. No final facade or callback namespace remains under `mz_core:server:inventory:*`.

The core still owns and validates:

- active Character snapshot and player-state actions;
- organization membership/authorization (`mz_org`);
- vehicle access;
- audit log persistence.

Inventory persists the authorized container but does not take ownership of those domains.

## Validation state

Static suites and syntax checks pass. Post-cutover FiveM/OneSync/MySQL validation is still required for A/B, A/B/A, hotbar, transfers, drops, reservations, consumers, disconnect/source reuse and restart.

```text
INVENTORY_WRITER_OWNER = mz_inventory
INVENTORY_SINGLE_WRITER = YES
INVENTORY_DOMAIN_EXTRACTION_READY = NO
READY_FOR_NEXT_DOMAIN_WAVE = NO
READY_FOR_ORG_EXTRACTION = NO
```
