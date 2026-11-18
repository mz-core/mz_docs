# Metadata persistence: deployment preparation

Status: LOCAL TESTING. CAS, session fencing and bounded in-memory reconciliation
are implemented; FiveM/MySQL runtime validation and durable crash recovery
remain pending. Do not deploy it as a complete persistence fix.

The barrier increments `mz_players.metadata_version` after verifying the
active session and reads the resulting metadata and exact decimal version in
the same transaction. Metadata writes compare account, character, citizenid
and the expected version, and increment the version atomically. A conflict or
unknown write result blocks additional flushes for that runtime; it does not
establish whether an uncertain write committed. Versions travel as decimal
strings, not Lua or JavaScript numbers. The barrier and CAS depend on all
metadata writers respecting the protocol.

The running resource reserves at most 128 active or pending character state
slots, rejects metadata over 65536 bytes at admission, and blocks admission when
the limit is reached. On an unconfirmed flush it retains the player metadata,
dirty fields, original session and attempted snapshot in memory, keyed by
character; further activation of that character is rejected. The internal
server-only `getPendingPlayerState(accountId, characterId)` returns a detached
copy. Automatic reconciliation locks the character row, checks the expected
version and exact submitted content, and writes the latest retained snapshot
with a version increment in the same transaction. An unknown original write
with later differences remains blocked when its origin cannot be proven. A
database state already identical to the retained state can be fenced without
claiming which operation committed. Unknown recovery results retry at most five
times with backoff; conflicts and exhausted retries remain retained and blocked.
Resource stop, restart or process failure destroys this memory; this is not
durable recovery.

The owner and writer of fundamental player metadata remain `mz_core`. The
prepared migration is `mz_core/sql/metadata_version.sql`; it is not executed by
the resource. It adds a non-null unsigned BIGINT with default zero and does not
update existing metadata. MySQL DDL may lock or rebuild the table depending on
the actual server version and table; verify this on a representative copy.

Before deployment, finish and test the entire fenced persistence protocol and
session cleanup, validate the migration on a disposable copy, and take and
verify a backup. Stop all old Core instances and any other metadata writers;
old code writes metadata without a version predicate. Apply the reviewed SQL
manually, then start only the complete new Core and check bootstrap, session
activation, persistence, and audit logs. A schema without the column fails
bootstrap with `schema_migration_required:mz_players.metadata_version`; an
incompatible type/default fails with `schema_incompatible`.

If migration or deployment fails, stop writers and investigate schema and
backup state before retrying. Returning to old Core code removes write fencing;
do not do so while delayed writes or concurrent sessions can still execute.
Keep the added column when reverting application code; dropping it is a
separate destructive operation and is not authorized by this document.

A version column does not preserve changes that never reached MySQL. Runtime
validation, session reuse and process crashes remain unresolved risks.
