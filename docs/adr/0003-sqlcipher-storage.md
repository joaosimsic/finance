# ADR 0003: SQLCipher storage

Status: accepted. Date: 2026-11-16.

## Context

The app needs encrypted local storage for raw Pluggy payloads, normalized domain
data, and derived analytics. `SECURITY.md` requires SQLCipher encryption at rest,
a DB key derived from the master key, and no plaintext database.

## Decision

- Use SQLite through `rusqlite` with the `bundled-sqlcipher` feature.
- Derive the DB key with HKDF-SHA256 from the master key.
- Open the database only after vault unlock.
- Close the connection on lock.
- Set `foreign_keys = ON` and `temp_store = MEMORY`.
- Enable SQLCipher memory security where supported.
- Keep database and sidecar files app-private.
- Verify Linux and Android builds in Phase 2 with a spike.

Schema layers:

1. Raw Pluggy JSON.
2. Normalized domain.
3. Derived analytics cache.

Rules:

- Money is integer cents.
- Raw payloads are append-only.
- Local writes are allowed; external writes are not.
- Derived data is recomputable and versioned by source hash.
- No plaintext DB, plaintext temp files, or unencrypted WAL.

## Consequences

- Cross-compiling SQLCipher for Android is a known risk and must be proven early.
- `rusqlite` is synchronous; database access runs off the UI thread.
- The DB lifecycle is tightly coupled to app lock.
- Backup uses a logical data export, not a copy of the SQLCipher file.
- Migrations must be tested with an encrypted database.

## Alternatives considered

- `sqlx` with SQLCipher: more async-friendly but extra build complexity; rejected
  for the first implementation.
- Plain SQLite plus field-level encryption: violates `SEC-23`; rejected.
- Copying the encrypted DB file as backup: unusable without vault material and
  couples backup to device secrets; rejected.
