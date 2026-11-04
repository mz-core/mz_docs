---
title: Economy direct extraction
status: CANONICAL
authority: LEVEL_2_ARCHITECTURE
owner: mz_economy
last_reviewed: 2026-10-08
applies_to: [mz_core, mz_economy, mz_org, financial_consumers]
related_contracts: [MZ-OWNERSHIP-v1, ADR-0002, POST_ECONOMY_SINGLE_WRITER_CUTOVER]
---

# Economy direct extraction

`POST_ECONOMY_SINGLE_WRITER_CUTOVER` transfers financial authority from `mz_core` to `mz_economy` in the pre-production baseline. There is no core facade, dual-read, dual-write or compatibility migration.

## Identity Account is not Financial Account

`mz_core.mz_accounts` is the internal Identity Account. It binds external identifiers, Character ownership and sessions. It contains no `wallet`, `bank` or `dirty` balance.

`mz_economy.mz_player_accounts` is the Financial Account for a canonical Character. Its persistent subject is `citizenid`; runtime `source` is never a persisted financial key.

## Ownership

`mz_economy` exclusively owns and writes:

- `mz_player_accounts`;
- `mz_org_accounts`;
- `mz_account_idempotency`;
- `mz_org_account_operations` financial receipts;
- `mz_economy_transactions` canonical ledger;
- `mz_economy_daily_player_stats` derived reporting.

`mz_org_account_transactions` was removed because it duplicated the canonical ledger. `mz_financial_outbox`, its dispatcher/admin surface and Economy receipt consumer were removed because balance and ledger now share one owner and one transaction; no independent consumer existed.

## Atomicity

Balance updates, all ledger legs and the optional idempotency result are composed into one `MySQL.transaction.await`. Runtime projections update only after commit. Organization commerce keeps a pending operation receipt, then atomically commits balance, applied receipt state and ledger.

## Dependency direction

Consumers (`mz_bank`, `mz_org`, `mz_fuel`, `mz_houses`, `mz_furniture`, `mz_clothing`, `mz_tatto`, `mz_transport`) call `mz_economy` directly. Economy depends on `mz_core` only for canonical Character identity, player-state authorization and fundamental audit. Core has no dependency on Economy for Identity, Character, Session or lifecycle.

## Runtime status

Static, syntax and offline harness evidence is recorded at the checkpoint. MySQL, FXServer and client runtime remain `NOT_RUN` until explicitly executed in staging.
