# ADR 0005: Analytics definitions

Status: accepted. Date: 2026-11-16.

## Context

Expense, income, and investment analysis must be deterministic, explainable, and
resistant to common Brazilian double-counting problems such as credit-card bill
payments and internal transfers. The Phase 6 milestone includes anomalies,
changepoints, seasonality, and forecasts.

## Decision

### Periods

- Reporting time zone: `America/Sao_Paulo`.
- Store UTC timestamps.
- Bucket by local calendar month.
- Label partial periods explicitly.

### Cash flow

```text
income_cents   = income inflows excluding transfers and investment redemptions
expenses_cents = expense outflows excluding transfers, credit-card payments,
                 and investment contributions
savings_cents  = income_cents - expenses_cents
savings_rate   = savings_cents / income_cents when income_cents > 0
```

Investment contributions are reported separately and may be added to an optional
"total savings including investments" figure. Credit-card purchases are counted
once. Bill payments are excluded from expenses.

### Categories

Precedence:

1. User override.
2. Local rule, highest priority first.
3. Pluggy category if available.
4. Merchant fallback.
5. Uncategorized.

Categories and rules are local only. No Pluggy metadata writes.

### Merchant normalization

Deterministic pipeline: trim, case-fold, remove processor prefixes and noise,
remove CNPJ and punctuation, apply alias table, apply user override. Raw
description is always preserved.

### Recurring payments

Minimum 3 occurrences. Cadence windows:

- monthly: 25-35 days
- quarterly: 80-100 days
- annual: 350-380 days

Confidence uses interval regularity and amount stability. Subscription candidates
are marked separately. User can confirm or dismiss locally.

### Budgets

Budget per category and period. Actual spend from categorized expenses excluding
transfers, credit-card payments, and investment contributions.

### Anomalies

Robust z-score using median and MAD. Minimum 6 periods. Flags high and low spend.
No automatic category or budget changes.

### Changepoints

CUSUM or documented equivalent. Minimum 12 periods. Detects sustained shifts, not
one-off spikes.

### Forecasts

Seasonal decomposition plus Holt-Winters or documented equivalent. Horizon 3-6
months. Intervals from residual variance. Minimum 24 months for seasonal forecast;
fallback to non-seasonal with lower confidence. No point forecast without interval
and confidence label.

### Net worth

Assets minus liabilities. Snapshot after sync and on unlock when data changed.
Completeness flag when accounts, balances, or valuations are missing. No fabricated
historical backfill.

### Investments

- Contributions, withdrawals, income, fees, and taxes classified separately.
- TWR based on valuation dates and cash-flow timing.
- XIRR/MWRR solved numerically with convergence reporting.
- FIFO cost basis for realized and unrealized P&L.
- Benchmarks: CDI, SELIC, IPCA, IBOV, USD/BRL, with series IDs verified before
  implementation.
- Risk metrics from daily returns with documented annualization.
- Non-BRL positions converted with PTAX when available, otherwise excluded with a
  visible label.

## Consequences

- Metrics are reproducible from the same inputs and clock.
- Money stays in integer cents; ratios may use decimal or floating point.
- Data quality problems are visible through confidence and completeness labels.
- Analytics changes require version bumps and cache invalidation.
- Phase 6 includes the full analytics scope before investments begin.

## Alternatives considered

- Treat bill payments as expenses: double counts credit-card spending; rejected.
- Treat transfers as income or expense: inflates both; rejected.
- Automatic category writes to Pluggy: forbidden by `SEC-05`; rejected.
- Forecast without intervals or history thresholds: misleading; rejected.
