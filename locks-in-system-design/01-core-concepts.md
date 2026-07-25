# Core concepts

A lock protects an **invariant**, not merely a line of code. For a booking system, the invariant might be `reserved + available = capacity` and `available >= 0`.

## Vocabulary

| Term | Meaning |
| --- | --- |
| Critical section | Work that conflicts if it runs concurrently |
| Contention | Multiple workers needing the same protected resource |
| Granularity | Scope of one lock: global, tenant, account, row, or object |
| Throughput | Completed work per unit of time |
| Latency | Time a caller waits, including lock wait time |
| Deadlock | Cyclic waiting; none of the workers can proceed |
| Starvation | A worker continually fails to obtain the lock |

## The critical-section rule

Only include the state transition itself. Do not hold a lock while calling another service, rendering a document, or waiting for user input.

```text
Bad:  lock -> call payment provider -> update order -> unlock
Good: call payment provider -> lock -> verify/update local state -> unlock
```

The second version still needs idempotency and a reconciliation path, but it does not block every order while an external provider is slow.

## Granularity

```mermaid
flowchart LR
  G[One global lock: simple, high contention] --> T[Per-tenant lock]
  T --> A[Per-account lock]
  A --> R[Per-record lock: more concurrency, more complexity]
```

Choose a key that matches the conflict domain. A per-account lock is better than a global transfer lock if two unrelated accounts may be updated independently. But a transfer touching accounts A and B needs a consistent strategy for both.

## Correctness versus capacity

Use an exclusive lock or transaction to preserve correctness. Use a semaphore or rate limiter to protect capacity. A semaphore permitting ten requests does not prove two requests will not update the same order incorrectly.

Next: [Mutexes](02-mutex.md)
