# Pessimistic locking

Pessimistic locking reserves conflicting database data before changing it, typically for the lifetime of a short transaction.

```sql
BEGIN;
SELECT available FROM inventory WHERE sku = :sku FOR UPDATE;
-- If available > 0, decrement and insert reservation
UPDATE inventory SET available = available - 1 WHERE sku = :sku;
COMMIT;
```

Other transactions that require an incompatible lock on that row wait, fail by timeout, or follow the database's configured behavior.

## Suitable cases

- High-contention, short allocations: seats, scarce inventory, or account balances.
- A business operation must read several related values and update them consistently.

## Transaction discipline

```mermaid
flowchart LR
  A[Begin transaction] --> B[Lock rows in fixed order]
  B --> C[Validate invariant]
  C --> D[Write changes]
  D --> E[Commit immediately]
```

- Do not call external APIs inside the transaction.
- Lock multiple rows in a stable order, e.g. ascending account ID.
- Configure lock and transaction timeouts.
- Retry deadlock-victim and serialization errors only when the operation is idempotent.

## Prefer a conditional update when possible

For a one-row transition, this is often more compact:

```sql
UPDATE inventory
SET available = available - 1
WHERE sku = :sku AND available > 0;
```

If it affects one row, the reservation can proceed; if it affects zero, stock is unavailable. Use a transaction when related records must be updated atomically.
