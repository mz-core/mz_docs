---
title: Organization Domain Extraction
status: IMPLEMENTED_STATIC_RUNTIME_RETEST_PENDING
authority: LEVEL_4_REFERENCE
owner: mz_org
last_reviewed: 2026-10-04
applies_to: [mz_org, mz_core, organization]
related_contracts: [ADR-0004, ADR-0019]
---

# Organization domain extraction

`POST_ORG_DOMAIN_EXTRACTION` transfers the complete Organization authority from `mz_core` to `mz_org`. There is no shadow reader, dual writer or legacy core facade.

## Before and after

```text
BEFORE                           AFTER
mz_core                          mz_core
  Organization schema/service     Account + Character + Session
  membership/grade/duty            global staff authorization

mz_org                          mz_org
  panel/orchestration              Organization schema/repository/service
                                    membership/grade/org permission/duty
                                    recruitment/goals/audit
                                    only Organization writer
```

## Identity and authority

- Account authority: `mz_core`, keyed by `account_id`.
- Character authority: `mz_core`, persistently keyed by `citizenid`; `mz_players.id` remains an internal technical key.
- Organization authority: `mz_org`, persistently keyed by `org_id`.
- Membership identity: `citizenid + org_id`; runtime `source` is never persisted as ownership.
- Staff/global authorization: `mz_core`. Organization authorization: `mz_org`. An Org permission never grants a staff permission.

Duty is persistent in `mz_player_orgs.duty`, is valid only for an active membership and is owned/written by `mz_org`. Source-indexed caches are cleared on disconnect, identity-session close and resource stop; Character switch reloads membership for the new `citizenid`.

## Schema and writer

`mz_org` exclusively creates and writes:

- `mz_org_types`
- `mz_orgs`
- `mz_org_grades`
- `mz_org_permissions`
- `mz_player_orgs`
- `mz_org_goals`
- `mz_org_recruitment`
- `mz_org_audit`

Tables named `mz_org_accounts`, their transactions and operations remain financial state under the current account/economy implementation. Vehicle rows remain vehicle state. Those services accept typed organization descriptors only after `mz_org` has validated existence, membership and capability; they do not read Organization tables.

## Public contracts

The canonical server surface includes organization lookup/listing, Character organizations, membership/grade/duty reads, Org permission checks, membership and grade mutations, leadership, recruitment/goals and readiness. Existing productive consumers were redirected to `mz_org`; the equivalent Organization exports were removed from `mz_core`.

`mz_org -> mz_core` is allowed for active Character/session, global staff authorization and fundamental logs/financial contracts. Vehicle contracts use `mz_org -> mz_vehicles`; neither `mz_core` nor `mz_vehicles` depends on `mz_org`.

Productive consumers cut over to `mz_org` are `mz_admin`, `mz_computer`, `mz_doors`, `mz_furniture`, `mz_garagem`, `mz_houses`, `mz_inventory`, `mz_medical`, `mz_org_activities`, `mz_phone`, `mz_realestate` and `mz_vehicles`. Each declares the dependency explicitly; no consumer reads or writes the canonical Organization tables directly.

## Startup and staging

Start `oxmysql`, `ox_lib`, `mz_core`, prerequisites of `mz_org`, then `mz_org`, followed by consumers. A full server restart is required for this cutover.

No destructive SQL is required by the implementation. If a clean disposable staging reset is deliberately chosen, stop FXServer and drop only the eight Organization-owned tables above; never drop account, player, inventory, economy, vehicle or property tables. `mz_org` recreates and seeds its schema on the next start.

The former default `staff` organization is not seeded. Global Staff authority remains account-scoped in `mz_core`; Organization authority remains Character-membership-scoped in `mz_org`.

## Validation state

Static ownership, namespace, dependency, syntax and contract checks are required at the checkpoint. FiveM/OneSync/MySQL runtime validation remains pending until performed on the VPS.

The first real `/org` attempt on 2026-10-04 returned `session_unavailable`: the session export router sent the Character-owned `GetPlayerSnapshot` request to `mz_org` instead of `mz_core`. The route is fixed and covered by a local executable regression; runtime status remains pending until the corrected `mz_org` is restarted and retested in FiveM/MySQL.

```text
ORG_WRITER_OWNER = mz_org
ORG_SINGLE_WRITER = YES
ORG_SCHEMA_OWNER = mz_org
ORG_RUNTIME_VALIDATED = NO
ORG_DOMAIN_EXTRACTION_READY = NO
```
