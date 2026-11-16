# Implementation Specification

Status: planning. Last updated: 2026-11-16.

This document defines how to implement the product described in
[main.md](../main.md) under the constraints in [SECURITY.md](../SECURITY.md). The
security document is binding. If this specification conflicts with it, the
security document wins.

Execution order and phase gates are in [ROADMAP.md](ROADMAP.md). Architecture
decisions are recorded in `docs/adr/`.

## 1. Scope

One person. Brazil. Local-first. Linux desktop first, Android after the recovery
gate. Read-only Open Finance access through Pluggy's free personal path: Meu Pluggy
plus Connector 200. No hosted backend. No accounts. No telemetry. No cloud sync.
No payment initiation.

In scope:

- Manual Item ID linking.
- Account, balance, transaction, credit card bill, and investment ingestion.
- Local encrypted storage.
- Expense, income, and investment analytics.
- Encrypted data-only backup and restore.
- CSV and JSON export.
- Android later with the same Rust core and React UI.

Out of scope:

- Multi-user or commercial operation.
- Hosted service or cloud sync.
- Payments, Pix, or transfers.
- Pluggy Connect widget.
- Pluggy metadata writes.
- Forced item refresh.
- Screen-scraping connectors.
- iOS in the first milestone.
- Tax filing automation.

## 2. Architecture

```text
React + TypeScript UI
        |
        | Tauri IPC: typed commands, no secrets returned
        v
src-tauri app layer
        |
        +-- bank-core        domain types, use cases, DTOs
        +-- bank-vault       key derivation, vault format, lock state
        +-- bank-storage     SQLCipher, migrations, repositories
        +-- bank-pluggy      only HTTP egress, typed read methods
        +-- bank-analytics   deterministic calculations
        +-- bank-backup      encrypted backup, export, restore
        +-- bank-platform    keychain, permissions, Android flags
```

Hard boundaries:

- Only `bank-pluggy` may contain an HTTP client.
- Only `src-tauri` exposes IPC commands.
- `bank-core` contains no network or platform code.
- `bank-storage` never sees a passphrase, API key, or Client Secret.
- `bank-pluggy` never sees SQL or frontend types.
- `bank-analytics` reads domain data through `bank-core` traits.
- Frontend never receives secret-bearing DTOs.

The raw HTTP client is private to `bank-pluggy`. No generic `request()` method
exists. No Tauri command accepts a URL or HTTP method.

## 3. Repository layout

```text
.
├── main.md
├── SECURITY.md
├── docs/
│   ├── ROADMAP.md
│   ├── IMPLEMENTATION_SPEC.md
│   └── adr/
├── src/                         React UI
│   ├── components/
│   ├── features/
│   ├── lib/
│   ├── routes/
│   └── styles/
├── src-tauri/                   Tauri shell and IPC commands
│   ├── capabilities/
│   ├── src/
│   │   ├── commands/
│   │   ├── state/
│   │   └── main.rs
│   └── tauri.conf.json
├── crates/
│   ├── bank-core/
│   ├── bank-vault/
│   ├── bank-storage/
│   ├── bank-pluggy/
│   ├── bank-analytics/
│   ├── bank-backup/
│   └── bank-platform/
├── tests/
│   ├── mock-pluggy/
│   └── fixtures/
├── scripts/
└── .github/workflows/
```

Naming rules:

- Rust crates use `bank_` in code and `bank-` in paths.
- TypeScript files use kebab-case except React components.
- React components use PascalCase.
- SQL tables and columns use snake_case.
- JSON IPC fields use camelCase.
- Rust DTOs use `#[serde(rename_all = "camelCase")]`.

## 4. Technology choices

| Area | Choice |
|---|---|
| Desktop shell | Tauri 2 |
| Backend | Rust, stable toolchain pinned in `rust-toolchain.toml` |
| Frontend | React, TypeScript, Vite |
| UI data fetching | TanStack Query |
| Routing | React Router |
| Charts | A library with accessible table fallback; final choice ADR in Phase 5 |
| Database | SQLite through `rusqlite` with `bundled-sqlcipher` |
| HTTP | `reqwest` with rustls, redirects disabled, only in `bank-pluggy` |
| Crypto | `argon2`, `hkdf`, `aes-gcm`, `rand`, `zeroize`, `secrecy` |
| Serialization | `serde` and `serde_json` |
| Time | `chrono` plus `chrono-tz` |
| Decimal quantities | `rust_decimal`; money remains integer cents |
| IPC type generation | `ts-rs` or manually kept DTOs checked by test |
| Rust tests | `cargo test`, `insta` snapshots, `proptest` where useful |
| Frontend tests | Vitest and Testing Library |
| Mock server | Axum or Wiremock in `tests/mock-pluggy` |

Dependencies must be pinned and lockfiles committed. No dependency may bring an
unused HTTP client into another crate.

## 5. Runtime flows

### 5.1 First run

1. Load app configuration.
2. Check keychain availability.
3. Create a 32-byte device secret in the OS keychain.
4. Prompt for app passphrase.
5. Generate a 32-byte master key.
6. Derive KEK from passphrase and device secret.
7. Wrap the master key.
8. Create the vault file with `0600` permissions.
9. Open the SQLCipher database with the derived DB key.
10. Prompt for Client ID and Client Secret.
11. Encrypt Client Secret under the master-derived key.
12. Show onboarding for manual Item ID entry.

No network request occurs before the vault and database are ready.

### 5.2 Normal start

1. Check keychain availability.
2. Read the device secret.
3. Show unlock screen.
4. Prompt for app passphrase.
5. Derive KEK and unwrap master key.
6. Open SQLCipher.
7. Decrypt Client Secret.
8. Exchange Client Secret for an API key.
9. Load local data.
10. Run due sync if unlocked and online.

### 5.3 Lock

1. Clear frontend query caches and sensitive view state.
2. Zeroize API key, Client Secret, DB key, and master key.
3. Close the SQLCipher connection.
4. Clear in-memory item IDs if they are considered sensitive in the current view.
5. Return to the lock screen.

Reopening requires the keychain device secret and app passphrase again.

### 5.4 Sync

1. Verify unlocked.
2. Ensure the API key is fresh; refresh through `POST /auth` if needed.
3. Fetch accounts for the item.
4. Fetch transactions with an overlap window.
5. Fetch bills.
6. Fetch investments and investment transactions.
7. Store raw responses.
8. Normalize and upsert.
9. Invalidate derived cache for affected periods.
10. Snapshot balances and net worth.
11. Record an audit summary.

### 5.5 Backup

1. Require unlock.
2. Prompt for backup passphrase.
3. Serialize normalized domain data and local analysis data.
4. Derive backup key with Argon2id.
5. Encrypt with AES-256-GCM.
6. Write to user-selected path with `0600` permissions.
7. Return metadata only: timestamp, size, schema version.

### 5.6 Restore

1. User selects backup file.
2. Prompt for backup passphrase.
3. Derive backup key and authenticate ciphertext.
4. Validate backup schema version.
5. Create a new vault with a new device secret and new app passphrase.
6. Create a new master key and new database.
7. Import domain and local analysis data.
8. Report success.
9. Require Client Secret re-entry before sync.

Restore never recovers the lost app passphrase and never silently resets it.

## 6. Vault and key hierarchy

### 6.1 Vault file

Location: app-private data directory. Linux permissions `0600`.

Logical schema:

```json
{
  "version": 1,
  "kdf": {
    "algorithm": "argon2id",
    "m_cost_kib": 65536,
    "t_cost": 3,
    "p_cost": 1,
    "salt_b64": "..."
  },
  "master_key_wrap": {
    "algorithm": "aes-256-gcm",
    "nonce_b64": "...",
    "ciphertext_b64": "..."
  },
  "client_secret_box": {
    "algorithm": "aes-256-gcm",
    "nonce_b64": "...",
    "ciphertext_b64": "..."
  },
  "created_at": "2026-11-16T12:00:00Z",
  "updated_at": "2026-11-16T12:00:00Z"
}
```

The vault contains no plaintext Client Secret, master key, DB key, device secret, or
passphrase.

### 6.2 Key derivation

```text
device_secret       = 32 random bytes from OS keychain
argon2_output       = Argon2id(passphrase, salt, m=65536 KiB, t=3, p=1)
kek_ikm             = argon2_output || device_secret
kek                 = HKDF-SHA256(kek_ikm, info="bank:v1:kek", length=32)
master_key          = AES-256-GCM-unwrap(kek, master_key_wrap)
db_key              = HKDF-SHA256(master_key, info="bank:v1:sqlcipher", length=32)
client_secret_key   = HKDF-SHA256(master_key, info="bank:v1:client-secret", length=32)
```

Rules:

- Argon2id minimum: 64 MiB memory, 3 iterations, 1 lane, 16-byte random salt.
- Store KDF parameters with the vault.
- Calibrate upward if needed. Upgrade by re-wrapping, not weakening old vaults.
- AES-256-GCM uses a fresh 12-byte random nonce per wrap.
- AAD includes the vault version and the field purpose.
- Never reuse a nonce with the same key.
- Secret types do not implement `Debug`, `Serialize`, or `Display`.
- Use redacted debug output only.
- Zeroize secrets on drop, lock, and exit.
- Passphrase is never stored.
- Passphrase input uses a password field and is cleared after submit.
- The app does not read the clipboard automatically.

### 6.3 Backoff

- Track failed unlock attempts in app-private state.
- Delay: 1s, 2s, 4s, then capped at 5 minutes.
- Reset after successful unlock.
- Never wipe data automatically from wrong passphrases.
- Backoff state contains no passphrase or derived key material.

### 6.4 Client configuration

- Client ID is non-secret and may be stored in app configuration.
- Client Secret is encrypted under the master key.
- `configure_client` accepts Client ID and Client Secret once.
- The Rust side moves the secret into a `SecretString`, uses it, and zeroizes it.
- The frontend clears the secret input after submit.
- The API key is persisted nowhere.

### 6.5 Keychain failure and rotation

- Keychain unavailable: refuse to start. No plaintext fallback, software key file,
  or passphrase-only mode.
- Rotated keychain device secret invalidates the old vault.
- Recovery from vault invalidation requires an encrypted backup or re-sync from
  Pluggy.
- Lost passphrase has no recovery. Wipe and re-sync or restore from backup.

## 7. Storage

### 7.1 Connection rules

- Use `rusqlite` with `bundled-sqlcipher`.
- Set the key through the SQLCipher API, never by string concatenation.
- `foreign_keys = ON`.
- `temp_store = MEMORY`.
- Enable SQLCipher memory security where supported.
- Keep database and sidecar files app-private.
- Close the connection on lock.
- No plaintext database, plaintext temp files, or unencrypted WAL.
- DB key is derived only from the master key.

### 7.2 Schema

```text
schema_migrations(
  version INTEGER PRIMARY KEY,
  applied_at TEXT NOT NULL
)

items(
  item_id TEXT PRIMARY KEY,
  connector_id INTEGER,
  institution_name TEXT,
  status TEXT NOT NULL,
  consent_expires_at TEXT,
  last_sync_at TEXT,
  created_at TEXT NOT NULL
)

accounts(
  id TEXT PRIMARY KEY,
  item_id TEXT NOT NULL REFERENCES items(item_id),
  type TEXT,
  subtype TEXT,
  name TEXT,
  number_masked TEXT,
  currency TEXT NOT NULL,
  balance_cents INTEGER,
  balance_updated_at TEXT,
  raw_payload_id INTEGER REFERENCES raw_payloads(id)
)

transactions(
  id TEXT PRIMARY KEY,
  account_id TEXT NOT NULL REFERENCES accounts(id),
  posted_at TEXT NOT NULL,
  authorized_at TEXT,
  amount_cents INTEGER NOT NULL,
  currency TEXT NOT NULL,
  description_raw TEXT,
  merchant_normalized TEXT,
  category_id TEXT REFERENCES categories(id),
  category_source TEXT,
  status TEXT,
  is_transfer INTEGER NOT NULL DEFAULT 0,
  is_credit_card_payment INTEGER NOT NULL DEFAULT 0,
  is_investment_flow INTEGER NOT NULL DEFAULT 0,
  raw_payload_id INTEGER REFERENCES raw_payloads(id),
  updated_at TEXT NOT NULL
)

credit_card_bills(
  id TEXT PRIMARY KEY,
  account_id TEXT NOT NULL REFERENCES accounts(id),
  due_date TEXT,
  close_date TEXT,
  total_cents INTEGER,
  minimum_cents INTEGER,
  status TEXT,
  raw_payload_id INTEGER REFERENCES raw_payloads(id)
)

investments(
  id TEXT PRIMARY KEY,
  item_id TEXT NOT NULL REFERENCES items(item_id),
  name TEXT NOT NULL,
  type TEXT,
  subtype TEXT,
  currency TEXT NOT NULL,
  quantity TEXT,
  unit_price_cents INTEGER,
  value_cents INTEGER,
  valuation_date TEXT,
  raw_payload_id INTEGER REFERENCES raw_payloads(id)
)

investment_transactions(
  id TEXT PRIMARY KEY,
  investment_id TEXT NOT NULL REFERENCES investments(id),
  trade_date TEXT NOT NULL,
  type TEXT NOT NULL,
  quantity TEXT,
  unit_price_cents INTEGER,
  amount_cents INTEGER,
  currency TEXT NOT NULL,
  raw_payload_id INTEGER REFERENCES raw_payloads(id)
)

categories(
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  parent_id TEXT REFERENCES categories(id),
  kind TEXT NOT NULL,
  is_system INTEGER NOT NULL DEFAULT 0
)

category_rules(
  id INTEGER PRIMARY KEY,
  priority INTEGER NOT NULL,
  matcher_json TEXT NOT NULL,
  category_id TEXT NOT NULL REFERENCES categories(id),
  enabled INTEGER NOT NULL DEFAULT 1
)

budgets(
  id INTEGER PRIMARY KEY,
  category_id TEXT NOT NULL REFERENCES categories(id),
  period_start TEXT NOT NULL,
  period_end TEXT NOT NULL,
  amount_cents INTEGER NOT NULL
)

recurring_series(
  id TEXT PRIMARY KEY,
  merchant_normalized TEXT NOT NULL,
  cadence TEXT NOT NULL,
  next_expected_date TEXT,
  typical_amount_cents INTEGER,
  confidence REAL NOT NULL,
  status TEXT NOT NULL
)

balance_snapshots(
  id INTEGER PRIMARY KEY,
  account_id TEXT NOT NULL REFERENCES accounts(id),
  snapshot_date TEXT NOT NULL,
  balance_cents INTEGER NOT NULL,
  source TEXT NOT NULL,
  UNIQUE(account_id, snapshot_date)
)

net_worth_snapshots(
  snapshot_date TEXT PRIMARY KEY,
  assets_cents INTEGER NOT NULL,
  liabilities_cents INTEGER NOT NULL,
  net_cents INTEGER NOT NULL,
  completeness TEXT NOT NULL
)

raw_payloads(
  id INTEGER PRIMARY KEY,
  endpoint TEXT NOT NULL,
  item_id TEXT,
  account_id TEXT,
  request_fingerprint TEXT NOT NULL,
  fetched_at TEXT NOT NULL,
  sha256 TEXT NOT NULL,
  body_json BLOB NOT NULL,
  UNIQUE(endpoint, request_fingerprint, sha256)
)

sync_runs(
  id TEXT PRIMARY KEY,
  item_id TEXT NOT NULL REFERENCES items(item_id),
  started_at TEXT NOT NULL,
  finished_at TEXT,
  status TEXT NOT NULL,
  cursor TEXT,
  requests_count INTEGER NOT NULL DEFAULT 0,
  error_code TEXT
)

audit_events(
  id INTEGER PRIMARY KEY,
  occurred_at TEXT NOT NULL,
  event_type TEXT NOT NULL,
  method TEXT,
  host TEXT,
  path_template TEXT,
  status_code INTEGER,
  detail TEXT
)

settings(
  key TEXT PRIMARY KEY,
  value_json TEXT NOT NULL
)
```

Indexes:

```text
transactions(account_id, posted_at)
transactions(category_id, posted_at)
transactions(merchant_normalized, posted_at)
credit_card_bills(account_id, due_date)
investment_transactions(investment_id, trade_date)
balance_snapshots(account_id, snapshot_date)
raw_payloads(endpoint, item_id, fetched_at)
sync_runs(item_id, started_at)
audit_events(occurred_at)
```

### 7.3 Data rules

- Money is integer cents. Never store money as float.
- Quantities and unit prices that are not currency use decimal text.
- Timestamps are UTC ISO 8601.
- Reporting dates use `America/Sao_Paulo`.
- Currency is uppercase ISO 4217.
- Transaction sign convention: negative is outflow, positive is inflow. The raw
  layer preserves Pluggy's original representation.
- Raw payloads are append-only.
- Local writes are allowed. External writes are forbidden.
- No CPF or identity data unless strictly required and approved.
- Audit events contain no tokens, bodies, account IDs, or passphrases.
- Do not hard-delete financial history for reconciliation. Mark local removals
  explicitly.
- Derived tables are recomputable and never the source of truth.

### 7.4 Derived cache

Derived results include:

- source dataset hash
- analytics version
- computed timestamp
- serialized result

Invalidate when source tables or category rules change. Analytics must be
deterministic for the same inputs and clock.

## 8. Read-only Pluggy client

### 8.1 Endpoint allowlist

Base URL is a fixed constant: `https://api.pluggy.ai`. Production host cannot be
overridden by environment variable, config file, or frontend input.

Allowed:

```text
POST /auth
GET  /accounts
GET  /accounts/{id}
GET  /accounts/{id}/balance
GET  /v2/transactions
GET  /transactions/{id}
GET  /bills
GET  /bills/{id}
GET  /investments
GET  /investments/{id}
GET  /investments/{id}/transactions
GET  /connectors
```

Forbidden:

- Any `PATCH`, `DELETE`, or other `POST`.
- `/payments`, `/pix`, `/webhooks`, `/categories/rules`, `/connect_token`.
- `PATCH /items`.
- Pluggy Connect widget.
- Generic request builder.
- Dynamic URLs or request methods.

`GET /items` is not in the allowlist. Do not add it without a `SECURITY.md` change.

### 8.2 Typed methods

```text
authenticate() -> ApiKeyLease
list_accounts(item_id) -> Vec<Account>
get_account(account_id) -> Account
get_balance(account_id) -> Balance
list_transactions(account_id, from, to, cursor) -> TransactionPage
get_transaction(transaction_id) -> Transaction
list_bills(account_id) -> Vec<Bill>
get_bill(bill_id) -> Bill
list_investments(item_id) -> Vec<Investment>
get_investment(investment_id) -> Investment
list_investment_transactions(investment_id) -> Vec<InvestmentTransaction>
list_connectors() -> Vec<Connector>
```

No public method accepts a raw path, query string, or method verb.

### 8.3 Authentication

- `POST /auth` JSON body includes `clientId` and `clientSecret`.
- Client Secret never appears in a URL, query string, log, or error.
- API key returned by Pluggy is held in a Rust-only `SecretString` or equivalent.
- API key lifetime is at most 2 hours.
- Zeroize on drop, lock, or exit.
- Never persist the API key.
- Refresh only through `POST /auth`.

### 8.4 Request rules

- Build URL from endpoint template plus encoded path segments.
- No secrets in URLs or query strings.
- API key goes in `X-API-KEY`.
- Disable redirects.
- TLS verification on. Never disable certificate validation.
- Apply connect and read timeouts.
- Egress allowlist: `api.pluggy.ai` and `api.bcb.gov.br`.
- Deny every other host before the request.
- Audit every outbound request: time, method, host, path template, status.
- Audit must not include tokens, bodies, query values, or account IDs.
- Any non-GET request is an alert event.
- Unknown endpoint or method is a hard error and audit event.

### 8.5 Error mapping

| Condition | Behavior |
|---|---|
| 401 | Re-authenticate once, then fail |
| 403 consent | Mark item as needing reauthentication |
| 404 | Return not found; do not retry |
| 429 | Honor `Retry-After`, then backoff with jitter |
| 5xx | Bounded retry with backoff |
| Timeout | Bounded retry |
| Network failure | Mark sync failed; keep local data |
| Invalid response schema | Fail sync with safe error code |

Logs contain safe error codes, not response bodies.

### 8.6 BCB benchmark client

- Host allowlist: `api.bcb.gov.br`.
- Typed series fetches only.
- Series IDs are verified in an ADR before implementation.
- No secrets are sent.
- Failure does not block core dashboards.
- Cache benchmark data locally with fetch date and source series.

## 9. Sync

### 9.1 Linking

- User enters Item ID manually.
- No Pluggy Connect widget.
- No `POST /connect_token`.
- Store Item ID, connector ID if known from user selection, institution name, and
  last sync metadata.
- No webhooks.
- Reauthentication happens in Meu Pluggy or the bank app; the app only shows the
  state.

### 9.2 Schedule

- Sync on unlock when due.
- At most once per 24 hours while app is running.
- Manual "fetch latest cached data" may issue allowed GETs only.
- No forced item refresh. No `PATCH /items`.
- No background Android service in the first milestone.
- Show `last_synced_at` and stale state when older than 24 hours.

### 9.3 Fetch procedure

1. `GET /accounts?itemId=...`
2. For each account, `GET /accounts/{id}/balance`.
3. For each account, `GET /v2/transactions` with overlap window.
4. Follow cursor until exhausted.
5. For credit accounts, `GET /bills?accountId=...`.
6. `GET /investments?itemId=...`.
7. For each investment, `GET /investments/{id}/transactions`.
8. Store raw payload for each response page.
9. Normalize and upsert.
10. Update item `last_sync_at`.
11. Record `sync_runs` summary.
12. Snapshot balances and net worth.

Transaction window:

- Initial sync: maximum allowed range by Pluggy, split into bounded windows.
- Incremental sync: previous sync date minus 7 days through today.
- Deduplicate by Pluggy transaction ID.
- Guard against repeated cursor values and maximum page count.
- If a response is malformed, do not partially normalize that page.

### 9.4 Normalization

- Preserve raw JSON before normalization.
- Map domain fields explicitly; do not pass raw JSON to frontend.
- Normalize amounts to signed cents with documented sign convention.
- Normalize status values to a local enum.
- Normalize dates to UTC and retain local reporting date.
- Compute hash for deduplication and raw payload identity.
- Classify transfers, credit-card payments, and investment flows.
- Do not write categories or metadata back to Pluggy.

### 9.5 Failure handling

- Failed item sync does not erase existing data.
- Store error code and user-safe message.
- 403 consent: mark as reauthentication required.
- 429: backoff and resume later.
- Partial sync: keep successfully committed pages; do not mark full success.
- UI shows last successful sync separately from last attempt.

## 10. Analytics

All analytics are deterministic, local, and recomputable. Money is integer cents.
Ratios use decimal or floating point only after integer aggregation.

### 10.1 Periods and time zone

- Primary reporting time zone: `America/Sao_Paulo`.
- Store UTC timestamps.
- Derive local date and month buckets.
- Month-over-month compares full calendar months.
- Year-over-year compares the same calendar month or period.
- Partial current month is labeled.

### 10.2 Cash flow and savings

Definitions:

```text
income_cents   = income inflows excluding transfers and investment redemptions
expenses_cents = expense outflows excluding transfers, credit-card payments,
                 and investment contributions
savings_cents  = income_cents - expenses_cents
savings_rate   = savings_cents / income_cents when income_cents > 0
```

Reporting:

- Show cash flow by month.
- Show investment contributions separately.
- Show optional "total savings including investments".
- Never double count credit-card purchases and the bill payment.
- Never double count internal transfers.
- Show uncategorized totals explicitly.
- Do not infer income from refunds or reversals without a rule.

### 10.3 Categories

Precedence:

1. User override on a transaction.
2. Local category rule, highest priority first.
3. Pluggy category if available.
4. Merchant fallback.
5. Uncategorized.

Rules:

- Categories are local. No Pluggy category-rule endpoints.
- System categories are seeded locally.
- User overrides apply locally only.
- Rule matching is deterministic and versioned.
- Recomputing analytics after a rule change invalidates affected cache.
- Category changes never modify external systems.

### 10.4 Merchant normalization

Pipeline:

1. Trim and case-fold.
2. Remove payment processor prefixes and noise tokens.
3. Remove CNPJ, punctuation, and repeated whitespace.
4. Apply deterministic merchant alias table.
5. Apply user override if present.
6. Preserve raw description.

Requirements:

- Deterministic for the same input.
- No external metadata writes.
- User aliases stored locally.
- Normalization version stored with derived results.
- Tests cover Brazilian payment processors and common punctuation.

### 10.5 Recurring payments and subscriptions

Detection:

- Minimum 3 occurrences for a series.
- Monthly window: 25-35 days.
- Quarterly window: 80-100 days.
- Annual window: 350-380 days.
- Amount tolerance: fixed-amount or bounded variation, documented per series.
- Confidence based on interval regularity and amount stability.
- Subscription candidate when recurring and merchant matches known subscription
  patterns or user marks it.
- User can confirm or dismiss a series locally.

Outputs:

- merchant
- cadence
- next expected date
- typical amount
- confidence
- status

### 10.6 Budgets

- User-defined budget per category and period.
- Actual spend from categorized expenses.
- Show remaining, percent used, and over-budget state.
- Exclude transfers, credit-card payments, and investment contributions.
- No external budget writes.

### 10.7 Anomaly detection

- Use robust z-score with median and MAD.
- Evaluate category and merchant monthly spend.
- Minimum history: 6 full periods.
- Flag both unusually high and unusually low spend.
- Never auto-change categories or budgets.
- Output includes score, baseline, observed amount, and confidence.
- Suppress duplicate flags for the same transaction set.

### 10.8 Changepoint detection

- Use CUSUM or a documented equivalent.
- Minimum history: 12 periods.
- Detect sustained level shifts, not one-off spikes.
- Output includes period, direction, magnitude, and confidence.
- No automatic user-facing financial advice.

### 10.9 Seasonality and forecasts

- Seasonal decomposition plus Holt-Winters or a documented equivalent.
- Forecast horizon: 3 to 6 months.
- Prediction intervals from residual variance.
- Minimum history: 24 months for seasonal forecast; fallback to non-seasonal with
  a lower confidence label if less data exists.
- Show confidence labels and data sufficiency.
- Never present a point forecast without an interval and label.
- Deterministic given the same data and clock.

### 10.10 Net worth

- Assets minus liabilities.
- Account balances and investment valuations included when available.
- Snapshot after sync and on unlock if data changed.
- Completeness flag when accounts, balances, or valuations are missing.
- No fabricated historical backfill.
- Historical snapshots are labeled as observed or estimated.

### 10.11 Investments

Positions:

- From `/investments`.
- Allocation by asset class, institution, and currency.
- Concentration metrics top-N and Herfindahl-Hirschman Index.

Cash flows:

- Contributions and withdrawals classified separately.
- Income, fees, and taxes classified from investment transaction types.
- Unknown types preserved and shown as unknown.

Returns:

- TWR based on valuation dates and cash-flow timing.
- XIRR/MWRR solved numerically with fallback and convergence error reporting.
- FIFO cost basis for realized P&L.
- Realized P&L from disposals using FIFO lots.
- Unrealized P&L = current value minus remaining FIFO cost.
- Fees and taxes tracked separately from principal.

Risk:

- Daily returns from valuation series.
- Volatility annualized with documented trading-day count.
- Sharpe and Sortino use a documented risk-free rate, defaulting to CDI when
  available.
- Maximum drawdown from cumulative return series.
- Minimum history thresholds and confidence labels.

Benchmarks:

- CDI, SELIC, IPCA, IBOV, USD/BRL.
- Series IDs verified in an ADR.
- Source host: `api.bcb.gov.br`.
- Benchmark failure does not block portfolio values.

Currency:

- Aggregate in BRL.
- Store original currency.
- Convert non-BRL with PTAX when available.
- Exclude from BRL aggregate with a visible label when conversion is unavailable.
- Never silently mix currencies.

## 11. IPC and frontend contract

### 11.1 Commands

```text
vault_status() -> VaultStatus
unlock(passphrase) -> UnlockOutcome
lock() -> ()
configure_client(client_id, client_secret) -> ()
add_item(item_id) -> ConnectionSummary
remove_local_item(item_id) -> ()
sync(item_id?) -> SyncSummary
list_accounts() -> AccountDto[]
list_transactions(filter) -> TransactionPage
list_bills(filter) -> BillDto[]
list_investments() -> InvestmentDto[]
get_dashboard(range) -> DashboardDto
create_backup(passphrase, path) -> BackupResult
restore_backup(passphrase, path) -> RestoreResult
export_data(format, path, range) -> ExportResult
```

Rules:

- `unlock` accepts a passphrase and returns no sensitive value.
- `configure_client` accepts a Client Secret once and zeroizes it after use.
- No command returns Client Secret, API key, master key, DB key, or device secret.
- No command accepts an arbitrary URL or HTTP method.
- Frontend never persists secrets.
- Passphrase input is cleared after submit.
- Clipboard is not read automatically.
- Errors use stable codes and safe messages.
- Local item removal is allowed. External deletion is not.
- No UI action triggers `PATCH /items`.
- Freshness and sync status are always visible.
- Frontend receives normalized domain DTOs, not raw Pluggy JSON.

### 11.2 Frontend state

Allowed persistence:

- Non-secret UI preferences.
- Selected date range.
- Theme.
- Last viewed screens.
- Non-secret filters.

Forbidden persistence:

- Passphrase.
- Client Secret.
- API key.
- Master key.
- DB key.
- Device secret.
- Raw credential material.
- Any field that can reconstruct a secret.

Use in-memory state for unlocked session data. Clear TanStack Query cache on lock.

### 11.3 UI screens

- Onboarding: create passphrase, configure Client ID and Client Secret.
- Lock and unlock.
- Connections: add Item ID, show consent state and last sync.
- Accounts: balances, freshness, account detail.
- Transactions: filters, categories, merchant, transfer flags.
- Bills: due dates, totals, status.
- Investments: positions, allocation, returns, risk, benchmarks.
- Dashboard: net worth, cash flow, spending, budgets, anomalies, forecasts.
- Settings: lock, backup, export, rotate Client Secret, local item removal.
- Backup and restore.

Required states:

- locked
- loading
- empty
- error
- stale
- reauthentication required
- partial data
- insufficient history

Accessibility:

- Keyboard navigation.
- Visible focus.
- Form labels and error announcements.
- Charts have accessible table alternatives.
- No color-only status.
- Touch targets sized for Android later.

Security UI:

- Android `FLAG_SECURE` on sensitive screens.
- Linux release builds disable devtools and core dumps.
- Strict CSP.
- Minimal Tauri capabilities.
- No remote content in the main webview.
- No remote Pluggy widget.

## 12. Backup, export, and recovery

### 12.1 Backup format

Data-only backup. No secrets.

Contents:

- Normalized domain data.
- Local categories, rules, budgets, merchant overrides, recurring user decisions.
- Settings excluding secrets.
- Schema version and app version.
- Export timestamp.
- Raw payloads excluded by default.

Excluded:

- Client Secret.
- API key.
- Master key.
- DB key.
- Device secret.
- Passphrase.
- Any keychain material.

Container:

```text
magic: "BANKBAK1"
version: 1
kdf: argon2id parameters and salt
cipher: aes-256-gcm
nonce
ciphertext
```

- Argon2id minimums match the vault.
- AES-256-GCM authenticates the whole payload.
- Atomic restore into a new database.
- Schema-version check before import.
- Tampered backup fails authentication.
- No backup stored in cloud sync by the app.

### 12.2 Restore

- Works on a clean device with the backup passphrase.
- Creates a new vault and new master key.
- Imports domain and local analysis data.
- Does not restore sync credentials.
- Prompts for Client Secret before future sync.
- Does not recover a lost app passphrase.
- No silent reset or recovery bypass.
- Test restore on a clean device as a release gate.

### 12.3 Export

- CSV and JSON.
- Domain data only.
- Selected date range and tables.
- No raw credentials, tokens, or secret fields.
- Redaction tests required.
- Export writes to user-selected path with `0600` permissions where supported.
- Warn that exports are plaintext and may contain financial data.

## 13. Security enforcement

| Control | Implementation |
|---|---|
| Read-only contract | Endpoint enum, typed methods, mock rejection |
| Single HTTP chokepoint | Only `bank-pluggy` has HTTP client |
| Egress allowlist | Fixed hosts and runtime validation |
| Secrets | `secrecy`, `zeroize`, redacted debug |
| Keychain | `SecretStore` trait, Linux then Android |
| DB encryption | SQLCipher, no plaintext fallback |
| Frontend | No secret DTOs, no browser persistence |
| Audit | Method, host, path template, status only |
| CI | Secret scan, forbidden endpoint scan, mock rejection |
| Release | Minimal capabilities, strict CSP, devtools off |

CI commands:

```text
cargo fmt --all -- --check
cargo clippy --workspace --all-targets -- -D warnings
cargo test --workspace
cargo audit
cargo vet
npm ci
npm run lint
npm run typecheck
npm test
gitleaks detect --source .
forbidden endpoint/path scan
tauri capability check
```

Forbidden strings for network-facing code:

```text
/payments
/pix
/webhooks
/categories/rules
connect_token
PATCH
DELETE
```

`PATCH` and `DELETE` scan scope is limited to `bank-pluggy` and network-facing
code. Local SQL and local delete operations are allowed under `SEC-06`.

## 14. Testing strategy

### 14.1 Unit tests

- vault derivation and tamper detection
- AES-GCM nonce handling
- passphrase backoff
- money helpers
- category precedence
- merchant normalization
- recurring detection
- anomaly and changepoint math
- forecast determinism and intervals
- FIFO lots and realized P&L
- return formulas
- date and time zone behavior

### 14.2 Integration tests

- SQLCipher open, close, and lock
- migrations
- repository upserts and deduplication
- raw payload retention
- sync cursor pagination
- partial sync recovery
- backup and restore
- export redaction
- keychain failure
- keychain rotation

### 14.3 Contract tests

- mock Pluggy server
- every allowed endpoint succeeds
- every forbidden method and path fails
- malformed responses
- 401, 403, 404, 429, 5xx
- cursor loop guard
- redirect rejection
- egress rejection
- BCB benchmark failure does not block core data

### 14.4 Property tests

- money integer arithmetic and overflow bounds
- idempotent sync
- category assignment determinism
- FIFO invariants
- backup round-trip authenticity

### 14.5 Security tests

- app refuses to start when keychain is unavailable
- wrong passphrase fails and triggers backoff
- tampered vault fails
- rotated keychain device secret invalidates vault
- database cannot open without master key
- backup restores on clean device with only the passphrase
- no secret in logs, exports, frontend storage, or crash output
- mock server rejects every non-allowed method and endpoint
- static scan finds no forbidden endpoint or verb
- app never sends a secret to the frontend
- any non-GET external request triggers an alert

### 14.6 Manual E2E checklist

1. Onboard on a clean Linux profile.
2. Lock and unlock.
3. Add an Item ID.
4. Sync accounts and transactions.
5. Browse dashboard, accounts, transactions, bills, investments.
6. Trigger a 403 consent state and verify UI.
7. Create backup.
8. Wipe app data.
9. Restore from backup with only backup passphrase.
10. Re-enter Client Secret.
11. Re-sync.
12. Verify no secrets in logs or files.

## 15. Build and release

- Pin Rust toolchain and dependencies; commit lockfiles.
- Run `cargo audit`, `cargo vet`, and `npm audit`.
- Secret scanning in pre-commit and CI.
- Signed release builds.
- No real credentials in CI.
- Minimal Tauri capabilities.
- Strict CSP.
- No remote content in main webview.
- Devtools disabled in release.
- Linux: require Secret Service; enable full-disk encryption; disable core dumps.
- Android: `FLAG_SECURE`, app-private storage, `allowBackup=false` for secret
  material, signed APK with no secrets.
- Release artifact contains no environment or CLI secret read path.

## 16. Definition of done

A phase is done only when:

- Its deliverables exist.
- Its tests pass in CI.
- Its SEC constraints are verified.
- No forbidden endpoint or secret leak is introduced.
- Documentation is updated.
- The next phase does not depend on an undocumented shortcut.

Release is done only when the full `SECURITY.md` verification matrix passes and
the restore and rotation drills are complete.
