# Security Hard Constraints

Status: binding. Last updated: 2026-11-16.

This document defines the non-negotiable security contract for the app. A release
that violates any constraint here must not ship.

## 1. Purpose and scope

The app is a local-first personal finance tool for one person in Brazil. It reads
accounts, transactions, credit card bills, and investments through Pluggy using
the free personal path: Meu Pluggy plus Connector 200.

Security goals:

- Bank credentials never reach the app or Pluggy. Consent happens in the bank app.
- External financial access is read-only. The app never initiates payments.
- Secrets are protected at rest and in memory.
- Data stays on the device. No hosted backend, no telemetry, no cloud sync.

No hardware key is used. No YubiKey, no TPM, no StrongBox requirement. The root of
trust is the OS keychain plus a user passphrase.

## 2. Threat model

In scope:

- Stolen disk, stolen powered-off device, leaked backups.
- Other local users on the same machine.
- Keychain compromise alone.
- Passphrase compromise alone.
- Accidental leaks through logs, screenshots, exports, frontend storage, or APK.
- Supply-chain mistakes and malicious dependencies.
- Regressions that turn read-only access into write access.

Out of scope:

- Malware that captures both keychain access and the passphrase.
- Keyloggers, root/admin compromise, compromised OS.
- A stolen device that is powered on and unlocked.
- Physical coercion.
- Malicious hardware or firmware.

Residual risk:

- The OS keychain is not a vault against malware running as the user.
- Passphrase strength matters. A weak passphrase weakens the whole design.
- Pluggy API keys are not read-only scoped. The read-only guarantee comes from
  no Payments product, read-only bank consent, and the app-level allowlist.
- A leaked Client Secret can read connected data. It cannot move money without a
  separate payment consent.

## 3. Read-only contract

`SEC-01` External financial access is read-only. Allowed requests:

- `POST /auth`
- `GET /accounts`
- `GET /accounts/{id}`
- `GET /accounts/{id}/balance`
- `GET /v2/transactions`
- `GET /transactions/{id}`
- `GET /bills`
- `GET /bills/{id}`
- `GET /investments`
- `GET /investments/{id}`
- `GET /investments/{id}/transactions`
- `GET /connectors`

Every other method and endpoint is forbidden. No other `POST`, no `PATCH`, no
`DELETE`.

`SEC-02` No Payments product. No Pix. No transfers. No payment consent.

`SEC-03` No Pluggy Connect widget. No `POST /connect_token`. Item IDs are entered
manually.

`SEC-04` No forced refresh. No `PATCH /items`. Rely on Pluggy daily auto-sync and
show data freshness in the UI.

`SEC-05` No Pluggy metadata writes. No `PATCH /transactions`, no category-rule
endpoints, no webhook CRUD.

`SEC-06` Read-only applies to external systems. Local SQLite writes are required
and allowed.

## 4. Secret hierarchy

```
OS keychain device secret (32 random bytes)
User passphrase
        |
        v
Argon2id(passphrase, salt)
        |
        v
HKDF-SHA256(Argon2id output || device secret)
        |
        v
KEK
        |
        v
AES-256-GCM unwrap
        |
        v
Master key (32 random bytes)
        |
        v
HKDF-SHA256
        |
        +-- SQLCipher DB key
        +-- Client Secret encryption key
```

Stored vault contents:

- version
- Argon2id parameters and salt
- AES-GCM nonce
- wrapped master key
- encrypted Client Secret

The vault contains no plaintext Client Secret, master key, DB key, or passphrase.

Unlock:

1. Read the device secret from the OS keychain.
2. Prompt for the app passphrase.
3. Derive the KEK and unwrap the master key.
4. Open the SQLCipher database and decrypt the Client Secret.
5. Exchange the Client Secret for an API key in Rust memory.

Lock:

- Zeroize the master key, DB key, Client Secret, and API key.
- Close the SQLCipher connection.
- Require keychain plus passphrase again to reopen.

## 5. Hard constraints

### Secrets and unlock

`SEC-07` Client Secret is encrypted under the master key. The keychain alone cannot
decrypt it.

`SEC-08` API key lives only in Rust memory, for at most 2 hours. Zeroize on drop,
lock, or exit. Never persist it.

`SEC-09` Frontend never receives secrets. No `localStorage`, `sessionStorage`,
IndexedDB, cookies, or persisted state for secrets or passphrases.

`SEC-10` No secrets in logs, crash reports, telemetry, screenshots, APK, JS bundle,
environment variables, CLI arguments, URLs, exports, or config files.

`SEC-11` Secret types do not implement `Debug`, `Serialize`, or `Display`. Use
redacted debug only.

`SEC-12` Passphrase is never stored. Use a password input, clear it after submit,
and do not read the clipboard automatically. Warn that clipboard history may retain
pasted secrets.

`SEC-13` Argon2id parameters: minimum 64 MiB memory, 3 iterations, 1 lane, 16-byte
random salt. Calibrate upward if needed and store parameters with the vault. Upgrade
by re-wrapping, not by weakening old vaults.

`SEC-14` AES-256-GCM with a fresh 12-byte random nonce per wrap. HKDF-SHA256 for key
separation. Never reuse a nonce with the same key.

`SEC-15` Unlock backoff after repeated wrong passphrases. Do not wipe data
automatically; avoid creating a denial-of-service trigger.

`SEC-16` Keychain unavailable: refuse to start. No plaintext fallback, no software
key file, no password-derived-only mode.

`SEC-17` App lock on launch and after idle. Lock clears secrets and closes the DB.

### Network

`SEC-18` Single HTTP chokepoint: `ReadOnlyPluggyClient` with typed read methods.
No generic `request()`, no exposed raw HTTP client, no dynamic URLs.

`SEC-19` Runtime method and path allowlist. An unknown request is a hard error and
an audit event.

`SEC-20` Disable redirects. TLS verification on. Never disable certificate
validation.

`SEC-21` Egress allowlist: `api.pluggy.ai` and `api.bcb.gov.br`. Deny everything
else.

`SEC-22` Never place secrets in URLs or query strings. Client Secret goes in the
`POST /auth` JSON body; API key goes in `X-API-KEY`.

### Storage and device

`SEC-23` SQLCipher encryption at rest. DB key derived from the master key. No
plaintext database.

`SEC-24` Vault file and database are app-private. On Linux use `0600`; on Android
use app-private storage and `allowBackup=false` for secret material.

`SEC-25` Store only data needed for analysis. Do not fetch or store identity/CPF
unless strictly required.

`SEC-26` Android: `FLAG_SECURE` on screens showing secrets or account data. No
screenshots or screen recording of those screens.

`SEC-27` Linux: require Secret Service. Enable full-disk encryption. Disable core
dumps for release builds.

### Build and supply chain

`SEC-28` Pin dependencies and commit lockfiles. Run `cargo audit`, `cargo vet`, and
`npm audit`.

`SEC-29` Secret scanning in pre-commit and CI. Fail on any detected credential.

`SEC-30` Signed release builds. No real credentials in CI. No secrets in APK.

`SEC-31` Minimal Tauri capabilities. Strict CSP. No remote content in the main
webview. Devtools disabled in release.

`SEC-32` No remote Pluggy widget. This removes a remote JavaScript surface.

### Detection and response

`SEC-33` Audit outbound requests: time, method, host, path template, status. No
tokens, bodies, or account IDs.

`SEC-34` Alert on any non-GET request. That indicates a read-only regression.

`SEC-35` Audit unlock and lock events. No passphrase, secret, or account data in the
log.

`SEC-36` Rotate the Client Secret on schedule and immediately on suspicion. Revoke
bank consents when needed. Create a new Pluggy application if in-place rotation is
not available.

`SEC-37` If the keychain or passphrase is compromised: lock the app, rotate the
Client Secret, revoke consents, restore from a clean backup or re-sync.

`SEC-38` Encrypted backup and restore must exist. Backups use a passphrase-derived
key and are never stored in cloud sync. Test restore on a clean device.

`SEC-39` Lost keychain plus no backup: re-sync from Pluggy. Lost passphrase: no
recovery; wipe and re-sync. Do not add a silent reset or recovery bypass.

## 6. Enforcement

- `bank-pluggy` owns authentication and HTTP. The raw client is private.
- Endpoint enum with allowed method and path template. No string-built URLs.
- `secrecy` and `zeroize` for secret values.
- Platform `SecretStore` trait for keychain access.
- One IPC command accepts passphrase or secret. It returns no sensitive value.
- CI static scan for forbidden strings: `/payments`, `/pix`, `/webhooks`,
  `/categories/rules`, `connect_token`, `PATCH`, `DELETE`.
- Mock Pluggy server fails any request outside the allowlist.
- Release build contains no read of secrets from environment variables or CLI
  arguments.

## 7. Verification

Required tests:

- App refuses to start when the keychain is unavailable.
- Wrong passphrase fails and triggers backoff.
- Tampered vault fails authentication.
- Rotated keychain device secret invalidates the old vault; restore path works.
- Database cannot open without the master key.
- Backup restores on a clean device with only the passphrase.
- No secret appears in logs, exports, frontend storage, or crash output.
- Mock server rejects every non-allowed method and endpoint.
- Static scan finds no forbidden endpoint or verb.
- App never sends a secret to the frontend.

## 8. Incident response

If a leak is suspected:

1. Lock the app and remove the keychain entry.
2. Revoke bank consents at the bank or Meu Pluggy.
3. Rotate the Client Secret or create a new Pluggy application.
4. Re-link Item IDs.
5. Inspect Pluggy usage for unexpected requests.
6. Restore from a clean encrypted backup or re-sync.
7. Record what leaked and update the checklist.

If the Client Secret is lost, it cannot be recovered. Create a new Pluggy
application and re-link Items.

## 9. Non-goals

- Multi-user or commercial operation.
- Hosted service or cloud sync.
- Payment initiation, Pix, transfers.
- Screen-scraping connectors.
- Hardware keys, YubiKey, TPM, or StrongBox requirements.
- Tax filing automation.
- iOS in the first milestone.

## 10. Change control

Any change that weakens these constraints requires an explicit decision, a written
risk note, and a review of this document. Security constraints are release-blocking.
