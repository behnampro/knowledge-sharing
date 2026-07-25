# Spinlocks

A spinlock repeatedly attempts to acquire ownership instead of putting the waiting thread to sleep.

```c
while (atomic_exchange(&locked, true) == true) {
  // spin
}
/* critical section: a few instructions only */
atomic_store(&locked, false);
```

## Why it can be fast

Sleeping and waking a thread can be more expensive than waiting a few CPU cycles. This makes spinlocks useful in kernels, runtimes, and specialized low-level code where the owner is expected to release almost immediately.

## Why application code should avoid it

```mermaid
flowchart LR
  A[Waiter spins] --> B[Consumes CPU]
  B --> C[Less CPU for lock owner]
  C --> D[Lock takes longer to release]
```

If a lock holder is descheduled, performs I/O, or waits for a dependency, spinning burns CPU until it resumes. In a single-core environment, naïve spinning can be especially harmful.

## Practical guidance

- Do not spin around HTTP, database, filesystem, or remote-cache work.
- Prefer runtime-provided mutexes; they often spin briefly and then park efficiently.
- Use bounded retries/backoff when polling a remote lock, never a tight loop.
