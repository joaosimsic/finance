# Bank — Personal Finance App (Brazil)

## Idea

A local-first personal finance app for one person in Brazil. It pulls accounts,
transactions, credit card bills, and investments through Open Finance, then
provides deep statistical analysis of expenses, income, and investments.

No hosted backend. No accounts. No telemetry. Data stays on the device.

## Data source

Pluggy, via the free personal path:

- Meu Pluggy portal + Connector 200
- Up to 5 active connections, same CPF
- Initial connections: Itaú, Nubank, Rico Investimentos
- Read-only: no Pix, no payments, no transfers
- Open Finance consent; revocable at the bank or Meu Pluggy
- Daily refresh; expect some data delay
- API: `/v2/transactions` with cursor; avoid deprecated `/transactions`
- Personal connector limits: no guaranteed webhooks, item IDs stored manually,
  reauthentication happens in Meu Pluggy

## Product direction

Desktop and Android, one shared codebase.

- Tauri 2
- Rust core for Pluggy sync, storage, normalization, analytics
- React + TypeScript UI
- Linux desktop first, Android in the same project
- iOS later; requires macOS/Xcode

Offline-first: reads and analysis work without internet. Sync requires internet.

## Storage

- SQLite with SQLCipher encryption at rest
- Key in OS keychain / Android Keystore
- Raw Pluggy JSON layer + normalized domain layer + derived analytics layer
- Money stored as integer cents, never floats
- Encrypted local backups and CSV/JSON export

## Analysis scope

Expenses and income:

- Cash flow, savings rate, category trends, MoM/YoY
- Merchant normalization, recurring payments, subscriptions
- Budget vs actual
- Anomaly detection and changepoint detection
- Seasonality and 3–6 month forecasts with intervals
- Daily net worth snapshots

Investments:

- Positions, allocation, concentration
- Contributions and withdrawals
- TWR, XIRR/MWRR, realized and unrealized P&L
- Income, fees, and taxes
- Benchmarks: CDI, SELIC, IPCA, IBOV, USD/BRL
- Volatility, Sharpe, Sortino, max drawdown
- Clear confidence labels where history is incomplete

## Security hard constraints

Canonical contract: [SECURITY.md](SECURITY.md).

- External access is read-only: `POST /auth` plus `GET` only
- No Payments, Pix, transfers, widget, forced refresh, or metadata writes
- Client Secret encrypted under a master key unlocked by OS keychain plus app passphrase
- API key only in Rust memory; no secrets in frontend, logs, exports, or APK
- SQLCipher at rest; app lock zeroizes keys and closes the DB
- Single HTTP chokepoint with method/path allowlist and egress allowlist
- CI gates block forbidden endpoints, leaked secrets, and dependency issues

## Non-goals

- Multi-user or commercial product
- Hosted service or cloud sync
- Payment initiation
- Screen-scraping connectors
- Tax filing automation
- iOS in the first milestone

## Status

Planning. No code yet.
