# Roadmap

Status: planning. Last updated: 2026-11-16.

Implementation roadmap for the local-first Brazilian personal finance app. Product
scope is in [main.md](../main.md). Security constraints are in
[SECURITY.md](../SECURITY.md). The binding security document wins over this plan.

## Goal

Build a Linux-first desktop app, then Android from the same codebase, that reads
accounts, transactions, credit card bills, and investments through the Pluggy
personal path and provides deep local analysis. No hosted backend. No telemetry.
No cloud sync. Read-only external access.

## Locked decisions

| Decision | Choice |
|---|---|
| Docs location | `docs/` plus `docs/adr/` |
| Storage | `rusqlite` with `bundled-sqlcipher` |
| Backup | Data only; never contains Client Secret or master key |
| Analytics MVP | Full Phase 6 scope including anomalies, changepoints, seasonality, forecasts |
| Android | Starts after the Phase 8 recovery gate |
| Investment cost basis | FIFO |
| Desktop first | Linux until Phase 8 passes |

## Delivery principles

- Security spine before any network access.
- Vertical slices with tests, not layer-only work.
- Every phase maps to constraints in `SECURITY.md`.
- No real credentials in CI.
- Only `bank-pluggy` may perform HTTP.
- One user, one device at a time, no multi-user assumptions.
- Desktop Linux is the reference implementation.
- Android reuses the Rust core and React UI, with platform-specific secret storage.
- Any change that weakens `SECURITY.md` requires a written risk note and review.

## Phase table

| Phase | Outcome | Depends on | Exit gate |
|---|---|---|---|
| 0 | Workspace, CI, ADRs, endpoint skeleton | none | Builds on Linux; CI green |
| 1 | Vault, keychain, unlock/lock | 0 | `SEC-07` through `SEC-17` pass |
| 2 | SQLCipher storage and schema | 1 | DB opens only after unlock |
| 3 | Typed read-only Pluggy client and mock | 1, 2 | `SEC-18` through `SEC-22` pass |
| 4 | Manual item linking and sync | 3 | Real read-only sync works |
| 5 | Desktop UI shell | 2, 4 | Usable app; no secret reaches frontend |
| 6 | Full expense and income analytics | 2, 5 | All Phase 6 analytics reproducible |
| 7 | Investment analytics | 2, 5 | Returns and benchmarks with labels |
| 8 | Backup, export, recovery | 2, 5 | Clean-device restore drill passes |
| 9 | Android target | 5, 8 | Android security parity |
| 10 | Release hardening | all | Full SEC verification matrix |

## Milestones

| Milestone | Phases | Meaning |
|---|---|---|
| M1 Security spine | 0-2 | Vault, lock, encrypted DB |
| M2 Read-only sync MVP | 3-5 | Real data through allowlisted client |
| M3 Expense analytics | 6 | Full spending analysis |
| M4 Investments | 7 | Portfolio metrics and benchmarks |
| M5 Recovery and Android | 8-9 | Data recoverable; Android parity |
| M6 Release | 10 | Security verification complete |

## Critical path

```text
Phase 0 -> Phase 1 -> Phase 2 -> Phase 3 -> Phase 4 -> Phase 5 -> Phase 6 -> Phase 7
                                                              \-> Phase 8 -> Phase 9
Phase 10 after all release-blocking gates pass
```

Do not start real Pluggy sync before Phases 1-3 pass. Do not add Android before the
vault, storage, and recovery paths are stable.

## Phase 0 - Foundation

Outcome: compilable workspace with security boundaries visible before features.

Deliverables:

- Tauri 2 app shell.
- Rust workspace crates: `bank-core`, `bank-vault`, `bank-storage`, `bank-pluggy`,
  `bank-analytics`, `bank-backup`, `bank-platform`.
- React + TypeScript + Vite frontend.
- CI for format, lint, typecheck, tests, dependency pinning, lockfiles.
- ADRs for runtime stack, vault format, SQLCipher storage, read-only client,
  analytics definitions, and FIFO cost basis.
- Endpoint enum skeleton with method and path template.
- Audit event schema.
- Secret-scanning policy and forbidden-endpoint scan.
- Mock Pluggy server placeholder.

Tasks:

1. Create workspace layout from `IMPLEMENTATION_SPEC.md`.
2. Pin toolchain and dependency versions; commit lockfiles.
3. Add `cargo fmt`, `cargo clippy`, `cargo test`, frontend lint, typecheck, tests.
4. Add secret scan and forbidden endpoint scan to CI.
5. Add architecture test that only `bank-pluggy` depends on an HTTP client.
6. Write initial ADRs.
7. Create `tests/mock-pluggy` crate with allowlist behavior stub.
8. Create `tests/fixtures` with synthetic Pluggy payloads.

Exit gate:

```text
cargo fmt --all -- --check
cargo clippy --workspace --all-targets -- -D warnings
cargo test --workspace
npm ci
npm run lint
npm run typecheck
npm test
gitleaks detect --source .
forbidden endpoint/path scan
tauri capability check
```

- CI green on Linux.
- No secrets in repository.
- No crate except `bank-pluggy` references an HTTP client.

## Phase 1 - Vault, keychain, app lock

Outcome: app creates and unlocks a vault without network or database.

Deliverables:

- `SecretStore` trait for the platform device secret.
- Linux Secret Service implementation.
- Vault file format and serialization.
- Argon2id to HKDF-SHA256 to KEK to AES-256-GCM unwrap.
- Master key generation and wrapping.
- Client Secret encryption under the master-derived key.
- Unlock, lock, idle lock, and zeroization.
- Wrong-passphrase backoff.
- Keychain-unavailable hard failure.
- Secret types with redacted debug only.
- Single IPC command accepts passphrase and returns no sensitive value.
- Lock screen UI.

Tasks:

1. Implement vault crypto primitives and test vectors.
2. Define vault file schema and AAD rules.
3. Implement Linux keychain access through Secret Service.
4. Implement unlock, lock, and idle lock state machine.
5. Implement backoff with app-private state and no data wipe.
6. Add IPC commands `vault_status`, `unlock`, `lock`, `configure_client`.
7. Add lock screen and passphrase input that clears after submit.
8. Add tests for tampered vault, wrong passphrase, missing keychain, rotated device
   secret, and zeroization expectations.

Exit gate:

- `SEC-07` through `SEC-17` verified.
- Tampered vault fails authentication.
- Wrong passphrase fails and triggers backoff.
- App refuses to start without keychain.
- No plaintext master key, DB key, Client Secret, or passphrase on disk.
- Frontend never receives a secret.

## Phase 2 - Encrypted storage

Outcome: SQLCipher database opened only from the unlocked master key.

Deliverables:

- SQLCipher integration with `rusqlite` and `bundled-sqlcipher`.
- DB key derived with HKDF-SHA256 from the master key.
- Schema migrations.
- Raw Pluggy JSON layer.
- Normalized domain layer.
- Derived analytics layer.
- Audit table.
- App-private file permissions.
- DB close and key zeroization on lock.
- SQLCipher build spike for Linux and Android.

Tasks:

1. Add `bank-storage` with connection lifecycle tied to lock state.
2. Implement migrations and schema from `IMPLEMENTATION_SPEC.md`.
3. Add repositories for items, accounts, transactions, bills, investments, and
   investment transactions.
4. Add raw payload store with SHA-256 and deduplication.
5. Add audit event repository.
6. Add app-private permissions and `0600` vault/DB files on Linux.
7. Add tests for open failure without master key, migration round-trip, lock close,
   and no plaintext temporary files.
8. Spike `bundled-sqlcipher` on Android before Phase 9.

Exit gate:

- DB cannot open without the master key.
- Lock closes the connection and clears DB key material.
- Migration round-trip works.
- Money fields are integer cents.
- No plaintext database or plaintext temp files.
- Android SQLCipher spike documented.

## Phase 3 - Read-only Pluggy client

Outcome: every external financial request goes through one typed client.

Deliverables:

- `ReadOnlyPluggyClient` with named methods only.
- No generic `request()`, no dynamic URLs, no exposed raw HTTP client.
- Method and path allowlist matching `SEC-01`.
- URL built only from path templates and encoded path segments.
- Redirects disabled; TLS verification enabled.
- Egress allowlist: `api.pluggy.ai` and `api.bcb.gov.br`.
- `POST /auth` body carries Client Secret; API key uses `X-API-KEY`.
- API key lives only in Rust memory, maximum 2 hours, zeroized on drop.
- Mock Pluggy server rejecting all non-allowed requests.
- Static CI scan for forbidden strings.
- Typed BCB benchmark client.

Tasks:

1. Implement endpoint enum and typed methods.
2. Implement authentication and API key lifecycle.
3. Implement request validation, audit event, and error mapping.
4. Implement retry policy for 429 and 5xx.
5. Implement mock server with allowlist and failure modes.
6. Add contract tests for every allowed endpoint and every forbidden request.
7. Add static scan for `/payments`, `/pix`, `/webhooks`, `/categories/rules`,
   `connect_token`, `PATCH`, and `DELETE` in network-facing code.
8. Add egress and TLS tests.

Exit gate:

- Unknown endpoint is a hard error and audit event.
- Mock rejects every forbidden method and path.
- No non-GET request can be constructed.
- `SEC-18` through `SEC-22`, `SEC-33`, and `SEC-34` pass.
- API key is never written to disk or frontend.

## Phase 4 - Manual item linking and sync

Outcome: user enters Item IDs manually; app fetches and stores read-only data.

Deliverables:

- Manual Item ID entry; no Pluggy Connect widget.
- Connector support for Itau, Nubank, and Rico.
- Account, balance, transaction, bill, investment, and investment-transaction
  fetches.
- `/v2/transactions` cursor pagination.
- Overlap window for updated transactions.
- Idempotent upserts.
- Raw payload retention with SHA-256.
- Sync run records.
- Freshness calculation and stale indicators.
- Retry and backoff for 429 and 5xx.
- Consent and reauthentication error handling.

Tasks:

1. Implement `add_item`, `remove_local_item`, and `sync` use cases.
2. Implement account and transaction sync with cursor guard.
3. Implement bills and investments sync.
4. Implement normalization and transfer classification inputs.
5. Implement raw payload retention before normalization.
6. Implement balance and net worth snapshots.
7. Add sync status UI data and freshness rules.
8. Add idempotency, pagination, failure, and consent tests.

Exit gate:

- End-to-end mock sync works.
- Duplicate sync does not duplicate transactions.
- Cursor loop terminates correctly.
- No forced refresh and no `PATCH /items`.
- No real credentials used in tests.
- Freshness and stale states visible.

## Phase 5 - Desktop UI shell

Outcome: usable local app for connecting, syncing, and browsing data.

Deliverables:

- Onboarding.
- Lock and unlock screen.
- Connections.
- Accounts.
- Transactions.
- Bills.
- Investments.
- Settings and security.
- Sync status and freshness.

Requirements:

- No secrets in frontend storage.
- Passphrase input cleared after submit.
- Strict CSP.
- Minimal Tauri capabilities.
- Devtools disabled in release.
- Empty, loading, error, locked, and stale states.
- Keyboard navigation and accessible labels.
- Charts must have accessible table alternatives.

Tasks:

1. Build app shell and route structure.
2. Build unlock and onboarding flows.
3. Build connections and manual item entry.
4. Build account, transaction, bill, and investment screens.
5. Build freshness and sync status components.
6. Add accessibility tests and frontend secret-storage checks.
7. Add CSP and Tauri capability verification.

Exit gate:

- Secret-flow audit passes.
- Frontend receives only domain DTOs.
- No secret in `localStorage`, `sessionStorage`, IndexedDB, cookies, or persisted
  state.
- App usable without internet after unlock and local data exists.

## Phase 6 - Full expense and income analytics

Outcome: complete expense and income analysis from normalized data.

Deliverables:

- Cash flow.
- Savings rate.
- Category trends.
- Month-over-month and year-over-year.
- Merchant normalization.
- Recurring payments and subscriptions.
- Budget vs actual.
- Anomaly detection.
- Changepoint detection.
- Seasonality and 3-6 month forecasts with intervals.
- Daily net worth snapshots.
- Confidence labels.

Tasks:

1. Implement date bucketing in `America/Sao_Paulo`.
2. Implement signed money and ratio helpers.
3. Implement transfer, credit-card-payment, and investment-flow classification.
4. Implement category precedence and local rules.
5. Implement merchant normalization with user overrides.
6. Implement recurring and subscription detection.
7. Implement budget vs actual.
8. Implement robust anomaly detection.
9. Implement changepoint detection.
10. Implement seasonality and forecasts with intervals.
11. Implement net worth snapshots and completeness labels.
12. Add deterministic fixture tests for all metrics.

Exit gate:

- Full Phase 6 scope shipped.
- Metrics reproducible from stored raw data.
- Money uses integer cents; floats only for ratios.
- Minimum-history and incomplete-data labels visible.
- Transfer and credit-card-payment double counting handled.
- Anomaly, changepoint, seasonality, and forecast tests pass.

## Phase 7 - Investment analytics

Outcome: portfolio analysis with explicit confidence and valuation limits.

Deliverables:

- Positions, allocation, and concentration.
- Contributions and withdrawals.
- Income, fees, and taxes.
- TWR and XIRR/MWRR.
- FIFO realized and unrealized P&L.
- Benchmarks: CDI, SELIC, IPCA, IBOV, USD/BRL.
- Volatility, Sharpe, Sortino, and maximum drawdown.
- Incomplete-history labels.

Tasks:

1. Implement position and allocation views.
2. Implement cash-flow classification.
3. Implement FIFO cost basis and realized P&L.
4. Implement TWR with valuation dates.
5. Implement XIRR/MWRR solver and fallback.
6. Implement BCB benchmark series mapping after ADR verification.
7. Implement risk metrics with documented annualization.
8. Implement non-BRL conversion or exclusion labels.
9. Add fixture tests for returns, P&L, and risk metrics.

Exit gate:

- Benchmark calls stay within egress allowlist.
- Return formulas have fixture-based tests.
- FIFO cost basis is covered by ADR and tests.
- Non-BRL handling is explicit.
- Insufficient history is labeled, not hidden.

## Phase 8 - Backup, export, recovery

Outcome: user can recover data and export without exposing secrets.

Deliverables:

- Encrypted data-only backup.
- Restore on clean device.
- CSV and JSON export.
- Secret redaction checks.
- Key rotation path.
- Lost-keychain recovery path.
- Lost-passphrase wipe and re-sync path.
- No cloud-sync backup.

Backup contents:

- Normalized domain data.
- Local analysis data: categories, rules, budgets, recurring overrides.
- Settings excluding secrets.
- Raw payloads excluded by default.
- No Client Secret, master key, API key, passphrase, or keychain device secret.

Tasks:

1. Implement backup container with Argon2id and AES-256-GCM.
2. Implement logical export and restore into a new database.
3. Implement clean-device restore flow with new vault and master key.
4. Implement Client Secret re-entry prompt after restore.
5. Implement CSV and JSON export with redaction tests.
6. Implement tamper detection.
7. Run full clean-device restore drill.

Exit gate:

- `SEC-38` and `SEC-39` pass.
- Tampered backup fails.
- Clean-device restore works with only the backup passphrase.
- Restored app requires Client Secret re-entry for future sync.
- Exports contain no secrets or raw credential material.
- No backup stored in cloud sync.

## Phase 9 - Android

Outcome: same product on Android with platform-specific secret storage.

Tasks:

- Android Keystore device secret.
- App-private storage.
- `allowBackup=false` for secret material.
- `FLAG_SECURE` on sensitive screens.
- Android lifecycle lock.
- Touch and mobile layouts.
- Signed APK with no secrets.
- Mirror Linux security tests.
- Verify SQLCipher build from Phase 2 spike.

Exit gate:

- Android security parity tests pass.
- No secret in APK, logs, screenshots, or backups.
- Lock behavior works across background and foreground.
- Nightly build uses no real credentials.

## Phase 10 - Release hardening

Outcome: release-blocking security verification complete.

Tasks:

- Full `SECURITY.md` verification matrix.
- `cargo audit`, `cargo vet`, and `npm audit`.
- Secret scanning in pre-commit and CI.
- Signed release builds.
- Minimal Tauri capabilities.
- Devtools disabled.
- Incident response runbook.
- Restore drill on clean device.
- Release checklist.

Exit gate:

- All release-blocking constraints pass.
- No known critical dependency issue.
- Restore and rotation drills completed.
- Release build contains no secret read from environment variables or CLI
  arguments.

## Implementation order

Suggested PR sequence:

1. Workspace, CI, lint, typecheck.
2. Endpoint enum and forbidden-request test skeleton.
3. Vault crypto primitives and vectors.
4. Linux keychain and vault file.
5. Unlock/lock IPC and lock screen.
6. SQLCipher open, close, and migrations.
7. Core schema and repositories.
8. Mock Pluggy server with allowlist.
9. Typed Pluggy client and API key lifecycle.
10. Auth, accounts, and transactions sync.
11. Bills and investments sync.
12. Connection and account UI.
13. Transaction list and freshness UI.
14. Expense analytics core.
15. Category rules, merchants, and recurring.
16. Budgets, anomalies, and changepoints.
17. Forecasts and net worth snapshots.
18. Investment positions and cash flows.
19. Returns, risk metrics, and benchmarks.
20. Backup, restore, and export.
21. Android platform layer.
22. Release hardening and verification.

## Risk register

| Risk | Impact | Mitigation |
|---|---|---|
| SQLCipher and Tauri Android complexity | Android slip | Phase 2 spike; verify in Phase 9 |
| Linux Secret Service unavailable | App cannot start | Test target distros early; `SEC-16` hard fail |
| Pluggy personal-path limits | Data freshness gaps | Freshness UI; no webhooks; manual reauth |
| Daily data delay | Stale dashboard | Show `last_synced_at` and stale state |
| Analytics data quality | Wrong conclusions | Confidence labels; raw-layer retention |
| Weak passphrase | Vault compromise | Strength guidance; no silent recovery |
| Secret-scanning false positives | CI friction | Synthetic fixtures; documented allowlist |
| Forbidden endpoint regression | Read-only violation | Mock rejection; static scan; CI gate |
| Non-BRL positions | Mixed-currency errors | Convert with PTAX or exclude with label |
| Transfer classification mistakes | Inflated expenses | User overrides; confidence labels |

## SEC traceability

| Area | Constraints |
|---|---|
| Read-only contract | `SEC-01` through `SEC-06` |
| Vault and secrets | `SEC-07` through `SEC-17` |
| Network | `SEC-18` through `SEC-22` |
| Storage and device | `SEC-23` through `SEC-27` |
| Build and supply chain | `SEC-28` through `SEC-32` |
| Detection and response | `SEC-33` through `SEC-39` |

## Change control

Changes to phase scope are allowed only if they do not weaken a security gate.
Any change that weakens `SECURITY.md` requires an explicit decision, a written risk
note, and a review of that document. Security constraints are release-blocking.
