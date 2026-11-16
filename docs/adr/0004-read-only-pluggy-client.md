# ADR 0004: Read-only Pluggy client

Status: accepted. Date: 2026-11-16.

## Context

`SECURITY.md` requires read-only external access, one HTTP chokepoint, runtime
method and path allowlists, disabled redirects, TLS verification, and an egress
allowlist. Pluggy API keys are not scoped read-only, so the app must enforce
read-only behavior itself.

## Decision

Use one typed `ReadOnlyPluggyClient` in `bank-pluggy`.

Allowed endpoints:

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

Rules:

- No generic `request()`.
- No public method accepts a method verb, URL, or raw path.
- URLs are built from endpoint templates and encoded path segments.
- Client Secret only in the `POST /auth` JSON body.
- API key only in `X-API-KEY`, held only in Rust memory for at most 2 hours.
- Redirects disabled; TLS verification on.
- Egress allowlist: `api.pluggy.ai` and `api.bcb.gov.br`.
- Audit method, host, path template, and status only.
- Any non-GET external request is an alert.
- Mock server rejects every forbidden method and path.
- CI scans network-facing code for forbidden strings.

No Pluggy Connect widget. No `POST /connect_token`. No Payments, Pix, transfers,
webhooks, category-rule endpoints, or `PATCH /items`.

`GET /items` is not in the allowlist and must not be added without changing
`SECURITY.md`.

## Consequences

- Item IDs are entered manually.
- Reauthentication happens outside the app in the bank app or Meu Pluggy.
- Daily freshness depends on Pluggy auto-sync.
- The app cannot force a refresh or write categories back.
- Every new endpoint requires a security review and `SECURITY.md` change.

## Alternatives considered

- SDK or generic HTTP wrapper: expands attack surface and bypasses typed control;
  rejected.
- Pluggy Connect widget: remote JavaScript surface and `connect_token` endpoint;
  rejected.
- Direct bank scraping: explicitly out of scope; rejected.
