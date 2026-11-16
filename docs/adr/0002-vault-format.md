# ADR 0002: Vault format

Status: accepted. Date: 2026-11-16.

## Context

The app must protect the Client Secret and database key at rest without a hosted
backend or hardware key. The root of trust is the OS keychain plus a user
passphrase. `SECURITY.md` requires Argon2id, HKDF-SHA256, AES-256-GCM, a keychain
device secret, and zeroization.

## Decision

Use a versioned vault file with:

- Argon2id parameters and salt.
- Wrapped master key.
- Encrypted Client Secret.
- Version and timestamps.

Key hierarchy:

```text
device_secret     = 32 random bytes in OS keychain
argon2_output     = Argon2id(passphrase, salt, m=65536 KiB, t=3, p=1)
kek               = HKDF-SHA256(argon2_output || device_secret,
                                info="bank:v1:kek")
master_key        = AES-256-GCM-unwrap(kek, wrapped_master_key)
db_key            = HKDF-SHA256(master_key, info="bank:v1:sqlcipher")
client_secret_key = HKDF-SHA256(master_key, info="bank:v1:client-secret")
```

Rules:

- AES-256-GCM uses a fresh 12-byte random nonce per wrap.
- AAD binds version and field purpose.
- KDF parameters are stored with the vault.
- Argon2id minimums: 64 MiB, 3 iterations, 1 lane, 16-byte salt.
- Secret types use `secrecy` and `zeroize`; no `Debug`, `Serialize`, or `Display`.
- Passphrase is never stored.
- Keychain unavailable means refuse to start.
- Wrong passphrases trigger backoff, never automatic wipe.
- Changing app passphrase re-wraps the master key.
- Rotating the keychain device secret invalidates the old vault.
- Backup is data-only and contains no vault material.

Vault file permissions are `0600` on Linux. Android uses app-private storage.

## Consequences

- Losing the keychain device secret requires backup restore or re-sync.
- Losing the passphrase has no recovery.
- Vault upgrades must re-wrap the master key and keep old vaults readable.
- Client Secret is protected by the master key, not directly by the passphrase.
- Backup restore creates a new vault and new master key.

## Alternatives considered

- Passphrase-only KDF: weaker against offline guessing; rejected.
- Keychain-only encryption: violates `SEC-07`; rejected.
- Hardware key or TPM: out of scope in `SECURITY.md`; rejected.
- Plaintext fallback or software key file: forbidden; rejected.
