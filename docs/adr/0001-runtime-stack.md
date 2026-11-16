# ADR 0001: Runtime stack

Status: accepted. Date: 2026-11-16.

## Context

The product is a local-first personal finance app for one person in Brazil. It needs
encrypted local storage, OS keychain access, a desktop UI, and an Android target
from one shared codebase. External access is read-only and must be confined to a
single Rust client.

## Decision

- Use Tauri 2 for desktop and Android packaging.
- Use Rust for the core, storage, Pluggy access, analytics, backup, and platform
  secret storage.
- Use React with TypeScript and Vite for the UI.
- Build Linux desktop first.
- Add Android after the Phase 8 recovery gate.
- Keep iOS out of the first milestone.

Crate boundaries:

```text
bank-core, bank-vault, bank-storage, bank-pluggy,
bank-analytics, bank-backup, bank-platform
```

Only `bank-pluggy` may contain an HTTP client. Only `src-tauri` exposes IPC
commands.

## Consequences

- One Rust core serves both Linux and Android.
- Tauri capability and CSP configuration must be minimal and reviewed.
- Android requires a platform-specific keychain adapter and `FLAG_SECURE` handling.
- SQLCipher must be verified on both Linux and Android early.
- The frontend never receives secrets or raw Pluggy payloads.
- No hosted backend or telemetry is introduced.

## Alternatives considered

- Electron: larger runtime and broader remote-content surface; rejected.
- Native Linux plus separate Android app: duplicated core and higher drift risk;
  rejected.
- Web app plus local server: adds a local network surface and secret handling risk;
  rejected.
