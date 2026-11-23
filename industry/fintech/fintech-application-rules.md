# Fintech Application Rules

## Financial Data Integrity
- All monetary values stored as integers (cents/smallest currency unit) — never floats
- Currency always stored alongside amount — never assume USD
- Immutable transaction ledger: insert-only, never update financial records
- Double-entry bookkeeping for all balance changes — debits = credits always
- Reconciliation jobs to verify ledger integrity daily

```typescript
// Correct money representation
interface Money {
  amount: number;   // In cents, integer only
  currency: "USD" | "EUR" | "GBP"; // ISO 4217
}

// Wrong
const balance = 99.99; // Float — never do this
// Right
const balance = { amount: 9999, currency: "USD" };
```

## Audit & Compliance
- Every state change has an immutable audit trail: who, what, when, why
- Audit logs stored separately from operational DB — different access controls
- PII handling: encrypt at rest, minimize access, log all access
- Data retention policies enforced at DB level — not just application code
- SOC 2 readiness from day 1: access controls, encryption, monitoring, incident response

## Transaction Safety
- Idempotency keys on all payment operations — network retries must be safe
- Distributed locks for concurrent operations on the same account
- Saga pattern for multi-step financial workflows (payment + inventory + fulfillment)
- Never process a transaction without recording its intent first (two-phase commit pattern)
- Timeout + reconciliation for async operations — never leave state uncertain

## Security Hardening
- End-to-end encryption for sensitive data in transit AND at rest