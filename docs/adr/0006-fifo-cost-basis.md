# ADR 0006: FIFO investment cost basis

Status: accepted. Date: 2026-11-16.

## Context

Investment analytics need realized and unrealized P&L. Pluggy investment
transactions provide dated buys and sells, but cost-basis policy is a local
analytics decision.

## Decision

Use FIFO for investment cost basis.

Rules:

- Lots are created from acquisition transactions in trade-date order.
- Disposals consume the oldest available lots first.
- Realized P&L per disposal = proceeds minus FIFO cost of consumed lots minus
  disposal fees and taxes.
- Unrealized P&L = current value minus remaining FIFO cost.
- Fees and taxes are tracked separately and also included in P&L where documented.
- Same-day transactions use stable ordering by trade time if available, then by
  transaction ID.
- Missing or ambiguous acquisition history produces an `incomplete_history` label.
- Corporate actions that cannot be mapped are preserved and labeled, not guessed.
- FIFO is applied per investment ID unless a documented grouping is added later.

The method is recorded with derived results and invalidated when investment
transactions change. Tests must cover partial lots, multiple sells, fees, taxes,
and missing history.

## Consequences

- Realized P&L is lot-specific and deterministic.
- Results depend on complete transaction history.
- Short sales and derivatives may need a later extension; not in the first scope.
- Switching methods later requires a new analytics version and clear user-facing
  labeling.

## Alternatives considered

- Average cost: simpler but loses lot-level realized P&L; rejected.
- Per-asset user-selected method: adds schema and UI complexity for the first
  milestone; rejected.
- LIFO: not chosen as the default for this product; rejected.
